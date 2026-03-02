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

## 2. Проверка DNS

Перед настройкой nginx убедитесь, что домен уже указывает на этот сервер (иначе выпуск сертификата и проверки не сработают):

```bash
dig +short coinexplorers.com
dig +short www.coinexplorers.com
```

IP должен совпадать с вашим сервером.

---

## 3. Настройка nginx для coinexplorers.com (сначала только HTTP)

Сертификат Let's Encrypt выдаётся только после того, как по домену на порту 80 доступен путь `/.well-known/acme-challenge/`. Поэтому **сначала** создаём конфиг для продакшен-домена и поднимаем только HTTP-блок (без SSL), затем получаем сертификат и уже потом включаем полный конфиг с HTTPS.

**Что должно попасть в прод-конфиг без потерь** (полная копия тестового, только замена домена):

- Три блока `server`: (1) HTTP → HTTPS, (2) HTTPS www → non-www, (3) основной HTTPS с локациями.
- В основном HTTPS-блоке все локации как в тесте:
  - `location ^~ /.well-known/acme-challenge/`
  - `location = /sitemap.xml` → редирект на `/api/sitemap.xml`
  - `location = /news` (с условием по `ref=university.mitosis.org`, **auth_basic**, proxy на 3000)
  - `location /` (**auth_basic**, proxy на 3000)
  - `location ^~ /strapi/` (proxy на 1337, все заголовки и таймауты)
  - `location = /api/sitemap.xml` (proxy на 3000)
  - `location ^~ /api/` (proxy на 8000, таймауты)
- Общие настройки: `client_max_body_size 65M`, `include /etc/letsencrypt/options-ssl-nginx.conf`, `ssl_dhparam`, те же `proxy_set_header` и таймауты.

### 3.1. Каталог для ACME-challenge (webroot)

```bash
sudo mkdir -p /var/www/_acme-challenge/coinexplorers.com
sudo chown -R www-data:www-data /var/www/_acme-challenge/coinexplorers.com
```

### 3.2. Временный конфиг: только HTTP (порт 80) для coinexplorers.com

Он нужен только чтобы certbot мог пройти проверку. Создайте файл:

```bash
sudo nano /etc/nginx/sites-available/coinexplorers.com-http-only.conf
```

Вставьте **только** один блок `server` (копия первого блока из тестового конфига, с заменой домена и webroot):

```nginx
# Временный HTTP-блок только для выпуска SSL (после получения сертификата заменим полным конфигом)
server {
    listen 80;
    listen [::]:80;
    server_name coinexplorers.com www.coinexplorers.com;

    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/_acme-challenge/coinexplorers.com;
        allow all;
    }

    location / {
        return 301 https://coinexplorers.com$request_uri;
    }
}
```

Включите только этот конфиг (полный с SSL пока не подключаем — сертификатов ещё нет):

```bash
sudo ln -sf /etc/nginx/sites-available/coinexplorers.com-http-only.conf \
            /etc/nginx/sites-enabled/coinexplorers.com.conf
```

Проверка и перезагрузка nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4. Выпуск SSL‑сертификатов для coinexplorers.com

После того как nginx слушает порт 80 по домену `coinexplorers.com` и отдаёт `/.well-known/acme-challenge/`:

```bash
sudo certbot certonly --webroot \
  -w /var/www/_acme-challenge/coinexplorers.com \
  -d coinexplorers.com \
  -d www.coinexplorers.com
```

Проверьте, что сертификаты появились:

```bash
sudo ls -la /etc/letsencrypt/live/coinexplorers.com
```

Должны быть файлы `fullchain.pem` и `privkey.pem`.

---

## 5. Полный конфиг nginx для coinexplorers.com (без потерь настроек)

Теперь подставляем **полную** копию тестового конфига с заменой только домена и путей к сертификату/webroot. Все локации, auth_basic, таймауты и заголовки сохраняются.

**5.1. Создать полный конфиг из тестового (одна замена по домену)**

```bash
sudo sed 's/coinexplorers\.tw1\.su/coinexplorers.com/g' \
  /etc/nginx/sites-available/coinexplorers.tw1.su.conf \
  | sudo tee /etc/nginx/sites-available/coinexplorers.com-full.conf > /dev/null
```

Так в прод попадёт весь конфиг тестового домена: все три `server`, все `location`, auth_basic, proxy на 3000/1337/8000, таймауты, `client_max_body_size` и т.д.

**5.2. Включить полный конфиг вместо временного**

```bash
sudo rm -f /etc/nginx/sites-enabled/coinexplorers.com.conf
sudo ln -s /etc/nginx/sites-available/coinexplorers.com-full.conf \
           /etc/nginx/sites-enabled/coinexplorers.com.conf
```

**5.3. Проверить и перезагрузить nginx**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

После этого продакшен-домен обслуживается по HTTP и HTTPS с теми же настройками, что и тестовый (включая basic auth, редиректы и прокси).

> Временный файл `coinexplorers.com-http-only.conf` можно оставить (для истории) или удалить — продление сертификата certbot будет использовать уже полный конфиг, в котором есть `location ^~ /.well-known/acme-challenge/`.

---

## 6. Обновление фронтенд `.env` под coinexplorers.com

Текущий файл `/srv/coinexplorers.com/client/.env` содержит:

- `NEXT_PUBLIC_API_ENDPOINT=https://coinexplorers.tw1.su/strapi`
- `NEXT_PUBLIC_APP_URL=https://coinexplorers.tw1.su`
- `NEXT_PUBLIC_APP_HOST=https://coinexplorers.tw1.su`
- `NEXT_PUBLIC_STRAPI_URL=https://coinexplorers.tw1.su`

**6.1. Открыть и отредактировать `.env`**

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

## 7. Обновление Strapi-конфига `server.js`

Файл: `/srv/coinexplorers.com/server/config/server.js`

Сейчас там в проде выставлен URL по умолчанию на тестовый домен:

```js
url: env("PUBLIC_URL", "https://coinexplorers.tw1.su"),
```

**7.1. Отредактировать конфиг**

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

## 8. Обновление `ecosystem.config.js` (pm2)

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

**8.1. Отредактировать PUBLIC_CLIENT_URL**

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

## 9. Перезапуск процессов pm2 с новыми настройками

После правок `.env`, `server.js` и `ecosystem.config.js` нужно перезапустить процессы под управлением `pm2`.

**9.1. Загрузить обновлённый конфиг в pm2**

```bash
cd /srv/coinexplorers.com
pm2 startOrReload ecosystem.config.js
```

**9.2. Проверить статус процессов**

```bash
pm2 status
pm2 logs --lines 50
```

Убедитесь, что процессы `strapi`, `api`, `frontend` запущены без ошибок.

---

## 10. Проверка работы сайта по новому домену

**10.1. Локальная проверка c сервера**

```bash
curl -I https://coinexplorers.com
curl -I https://coinexplorers.com/api/health || curl -I https://coinexplorers.com/api
curl -I https://coinexplorers.com/strapi
```

Должны приходить коды `200`/`301` (в зависимости от маршрута), без 4xx/5xx.

**10.2. Проверка из браузера**

Откройте в браузере:

- `https://coinexplorers.com`
- `https://www.coinexplorers.com` (должно редиректить на без `www`)
- разделы сайта, критичные для SEO (категории, страницы, поиск).

---

## 11. Плавное отключение тестового домена (опционально)

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

## 12. Чек-лист перед завершением работ

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

