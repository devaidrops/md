## Универсальный план: тестовый домен → продакшен

**Цель**: по этому документу можно перенастроить сервер с любого тестового домена на любой продакшен-домен. В начале задаются переменные — дальше все команды от них не зависят.

Предполагается:

- фронтенд слушает `127.0.0.1:3000`, Strapi `127.0.0.1:1337`, API `127.0.0.1:8000`;
- используется `pm2` с конфигом `ecosystem.config.js`;
- в nginx уже есть рабочий конфиг для тестового домена (имя файла: `$TEST_DOMAIN.conf`).

---

## 0. Переменные (задать один раз в начале)

Откройте терминал и выполните блок ниже, **подставив свои значения**. Дальше все команды в документе используют эти переменные — можно копировать блоки подряд.

```bash
# Тестовый (текущий) домен — для него уже есть конфиг nginx в sites-available
export TEST_DOMAIN="coinexplorers.tw1.su"

# Целевой продакшен-домен
export PROD_DOMAIN="coinexplorers.com"

# Путь к проекту на сервере (каталог, в котором лежат client/, server/, ecosystem.config.js)
export PROJECT_DIR="/srv/coinexplorers.com"

# Каталог для бэкапов (опционально, по умолчанию ~/backup-coinexplorers)
export BACKUP_DIR="${BACKUP_DIR:-$HOME/backup-coinexplorers}"
```

Проверка (должны вывестись заданные значения):

```bash
echo "TEST_DOMAIN=$TEST_DOMAIN"
echo "PROD_DOMAIN=$PROD_DOMAIN"
echo "PROJECT_DIR=$PROJECT_DIR"
```

---

## 1. Предварительная проверка и бэкапы

**1.1. Проверить, что вы на нужном сервере**

```bash
hostname
ls -la "$PROJECT_DIR"
```

Убедитесь, что вывод соответствует прод-серверу и проекту.

**1.2. Проверить состояние nginx и pm2**

```bash
sudo systemctl status nginx
pm2 status
```

**1.3. Сделать бэкап ключевых конфигов**

```bash
mkdir -p "$BACKUP_DIR"

sudo cp "/etc/nginx/sites-available/${TEST_DOMAIN}.conf" \
        "$BACKUP_DIR/${TEST_DOMAIN}.conf.$(date +%F-%H%M%S)"

cp "$PROJECT_DIR/client/.env" \
   "$BACKUP_DIR/client.env.$(date +%F-%H%M%S)"

cp "$PROJECT_DIR/ecosystem.config.js" \
   "$BACKUP_DIR/ecosystem.config.js.$(date +%F-%H%M%S)"

cp "$PROJECT_DIR/server/config/server.js" \
   "$BACKUP_DIR/strapi.server.js.$(date +%F-%H%M%S)"
```

---

## 2. Проверка DNS

Перед настройкой nginx убедитесь, что продакшен-домен уже указывает на этот сервер:

```bash
dig +short "$PROD_DOMAIN"
dig +short "www.$PROD_DOMAIN"
```

IP должен совпадать с вашим сервером.

---

## 3. Настройка nginx для продакшен-домена (сначала только HTTP)

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
sudo mkdir -p "/var/www/_acme-challenge/$PROD_DOMAIN"
sudo chown -R www-data:www-data "/var/www/_acme-challenge/$PROD_DOMAIN"
```

### 3.2. Временный конфиг: только HTTP (порт 80) для продакшен-домена

Он нужен только чтобы certbot мог пройти проверку. Создаётся из переменных одной командой:

```bash
sudo tee "/etc/nginx/sites-available/${PROD_DOMAIN}-http-only.conf" << EOF
# Временный HTTP-блок только для выпуска SSL (после получения сертификата заменим полным конфигом)
server {
    listen 80;
    listen [::]:80;
    server_name $PROD_DOMAIN www.$PROD_DOMAIN;

    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/_acme-challenge/$PROD_DOMAIN;
        allow all;
    }

    location / {
        return 301 https://$PROD_DOMAIN\$request_uri;
    }
}
EOF
```

Включите только этот конфиг (полный с SSL пока не подключаем — сертификатов ещё нет):

```bash
sudo ln -sf "/etc/nginx/sites-available/${PROD_DOMAIN}-http-only.conf" \
            "/etc/nginx/sites-enabled/${PROD_DOMAIN}.conf"
```

Проверка и перезагрузка nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4. Выпуск SSL‑сертификатов для продакшен-домена

После того как nginx слушает порт 80 по домену `$PROD_DOMAIN` и отдаёт `/.well-known/acme-challenge/`:

```bash
sudo certbot certonly --webroot \
  -w "/var/www/_acme-challenge/$PROD_DOMAIN" \
  -d "$PROD_DOMAIN" \
  -d "www.$PROD_DOMAIN"
```

Проверьте, что сертификаты появились:

```bash
sudo ls -la "/etc/letsencrypt/live/$PROD_DOMAIN"
```

Должны быть файлы `fullchain.pem` и `privkey.pem`.

---

## 5. Полный конфиг nginx для продакшен-домена (без потерь настроек)

Подставляем **полную** копию тестового конфига с заменой только домена (все локации, auth_basic, таймауты сохраняются).

**5.1. Создать полный конфиг из тестового (замена тестового домена на продакшен)**

```bash
# Экранируем точку в тестовом домене для sed
TEST_ESC="${TEST_DOMAIN//./\\.}"
sudo sed "s/${TEST_ESC}/${PROD_DOMAIN}/g" \
  "/etc/nginx/sites-available/${TEST_DOMAIN}.conf" \
  | sudo tee "/etc/nginx/sites-available/${PROD_DOMAIN}-full.conf" > /dev/null
```

В прод попадёт весь конфиг тестового домена: все три `server`, все `location`, auth_basic, proxy на 3000/1337/8000, таймауты, `client_max_body_size` и т.д.

**5.2. Включить полный конфиг вместо временного**

```bash
sudo rm -f "/etc/nginx/sites-enabled/${PROD_DOMAIN}.conf"
sudo ln -s "/etc/nginx/sites-available/${PROD_DOMAIN}-full.conf" \
           "/etc/nginx/sites-enabled/${PROD_DOMAIN}.conf"
```

**5.3. Проверить и перезагрузить nginx**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

После этого продакшен-домен обслуживается по HTTP и HTTPS с теми же настройками, что и тестовый (включая basic auth, редиректы и прокси).

> Временный файл `${PROD_DOMAIN}-http-only.conf` можно оставить или удалить — продление сертификата certbot будет использовать полный конфиг, в котором уже есть `location ^~ /.well-known/acme-challenge/`.

---

## 6. Обновление фронтенд `.env` под продакшен-домен

В `$PROJECT_DIR/client/.env` нужно заменить тестовый домен на `$PROD_DOMAIN`.

**6.1. Подставить продакшен-домен в `.env` (одной командой)**

```bash
sed -i "s|https://${TEST_DOMAIN}|https://${PROD_DOMAIN}|g" "$PROJECT_DIR/client/.env"
sed -i "s|http://${TEST_DOMAIN}|https://${PROD_DOMAIN}|g" "$PROJECT_DIR/client/.env"
```

Проверка содержимого:

```bash
grep -E 'NEXT_PUBLIC_(API_ENDPOINT|APP_URL|APP_HOST|STRAPI_URL)' "$PROJECT_DIR/client/.env"
```

Все URL должны указывать на `https://$PROD_DOMAIN`. При необходимости допишите вручную недостающие переменные (например `NEXT_PUBLIC_POSTS_PRIORITY`, `NEXT_PUBLIC_REVIEWS_PRIORITY`).

---

## 7. Обновление Strapi-конфига `server.js`

В `server/config/server.js` в проде используется URL по умолчанию — меняем тестовый домен на продакшен.

**7.1. Заменить дефолтный URL в server.js**

```bash
sed -i "s|https://${TEST_DOMAIN}|https://${PROD_DOMAIN}|g" "$PROJECT_DIR/server/config/server.js"
grep -n "url:" "$PROJECT_DIR/server/config/server.js"
```

Должна быть строка вида `url: env("PUBLIC_URL", "https://$PROD_DOMAIN"),`. Остальные настройки (порт 1337, admin) не трогаем.

---

## 8. Обновление `ecosystem.config.js` (pm2)

В `ecosystem.config.js` в приложении `strapi` задаётся `PUBLIC_CLIENT_URL` — подставляем продакшен-домен.

**8.1. Заменить PUBLIC_CLIENT_URL**

```bash
sed -i "s|https://${TEST_DOMAIN}|https://${PROD_DOMAIN}|g" "$PROJECT_DIR/ecosystem.config.js"
grep -A1 "PUBLIC_CLIENT_URL" "$PROJECT_DIR/ecosystem.config.js"
```

Должно быть `PUBLIC_CLIENT_URL: "https://$PROD_DOMAIN"`. Остальные поля не меняем.

---

## 9. Перезапуск процессов pm2 с новыми настройками

После правок `.env`, `server.js` и `ecosystem.config.js` нужно перезапустить процессы под управлением `pm2`.

**9.1. Загрузить обновлённый конфиг в pm2**

```bash
cd "$PROJECT_DIR"
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
curl -I "https://$PROD_DOMAIN"
curl -I "https://$PROD_DOMAIN/api/health" || curl -I "https://$PROD_DOMAIN/api"
curl -I "https://$PROD_DOMAIN/strapi"
```

Должны приходить коды `200`/`301` (в зависимости от маршрута), без 4xx/5xx.

**10.2. Проверка из браузера**

Откройте в браузере:

- `https://$PROD_DOMAIN`
- `https://www.$PROD_DOMAIN` (должно редиректить на без `www`)
- разделы сайта, критичные для SEO (категории, страницы, поиск).

---

## 11. Плавное отключение тестового домена (опционально)

После того как убедились, что `$PROD_DOMAIN` работает стабильно и DNS перенесён:

### Вариант A: Оставить тестовый домен как есть

Оставить конфиг `$TEST_DOMAIN.conf` — трафик по старому домену продолжит работать (например, для внутренних тестов).

### Вариант B: 301‑редирект с тестового на продакшен

Подставить в конфиге тестового домена редирект на `$PROD_DOMAIN`:

```bash
sudo sed -i "s|return 301 https://${TEST_DOMAIN}|return 301 https://${PROD_DOMAIN}|g" "/etc/nginx/sites-available/${TEST_DOMAIN}.conf"
sudo nginx -t
sudo systemctl reload nginx
```

### Вариант C: Полностью отключить тестовый домен

Когда тестовый домен больше не нужен:

```bash
sudo rm "/etc/nginx/sites-enabled/${TEST_DOMAIN}.conf"
sudo nginx -t
sudo systemctl reload nginx
```

> Конфиг в `sites-available` и сертификаты тестового домена можно удалить позже, когда откат не потребуется.

---

## 12. Чек-лист перед завершением работ

- **DNS**: A/AAAA‑записи `$PROD_DOMAIN` и `www.$PROD_DOMAIN` указывают на этот сервер;
- **Сертификаты**: в `/etc/letsencrypt/live/$PROD_DOMAIN` есть актуальные `fullchain.pem` и `privkey.pem`;
- **nginx**:
  - есть конфиг `/etc/nginx/sites-available/${PROD_DOMAIN}-full.conf` и symlink в `sites-enabled`;
  - `nginx -t` проходит без ошибок;
  - редиректы HTTP→HTTPS и www→non-www работают;
- **Backend**: в `ecosystem.config.js` и `server/config/server.js` указан `$PROD_DOMAIN`; процессы `strapi`, `api`, `frontend` в `pm2 status` **online**;
- **Frontend**: в `client/.env` все URL ведут на `https://$PROD_DOMAIN`; страницы и sitemap отдают корректные коды;
- **Тестовый домен**: либо редиректит на продакшен (301), либо отключён по необходимости.

Если все пункты выполнены — перенастройка с `$TEST_DOMAIN` на `$PROD_DOMAIN` завершена.

