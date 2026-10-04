# Трекер продуктивности (PWA)

```
tracker/index.html     ~10.4 КБ (≈4.7 КБ в gzip) — весь UI, стили и JS
tracker/manifest.json  ~0.7 КБ
nginx/tracker.conf     конфиг сервера с Basic Auth
```

## 1. Файлы на сервер

```bash
sudo mkdir -p /var/www/tracker
sudo cp tracker/index.html tracker/manifest.json /var/www/tracker/
sudo chown -R root:root /var/www/tracker && sudo chmod 644 /var/www/tracker/*
```

## 2. Генерация .htpasswd

**Вариант А — `htpasswd` (bcrypt, рекомендуется):**

```bash
sudo apt install apache2-utils          # Debian/Ubuntu   (RHEL/Alma: sudo dnf install httpd-tools)
sudo htpasswd -c -B -C 12 /etc/nginx/.htpasswd-tracker myuser   # -c только при первом создании!
# второй пользователь / смена пароля — без -c:
sudo htpasswd -B -C 12 /etc/nginx/.htpasswd-tracker myuser
# удалить пользователя:
sudo htpasswd -D /etc/nginx/.htpasswd-tracker myuser
```

**Вариант Б — без установки пакетов (`openssl`, apr1-MD5):**

```bash
printf 'myuser:%s\n' "$(openssl passwd -apr1)" | sudo tee /etc/nginx/.htpasswd-tracker
```

Права доступа (читать может только nginx):

```bash
sudo chown root:www-data /etc/nginx/.htpasswd-tracker   # на RHEL группа — nginx
sudo chmod 640 /etc/nginx/.htpasswd-tracker
```

Файл лежит вне `root` сайта, поэтому скачать его по HTTP нельзя.

## 3. Nginx

```bash
sudo cp nginx/tracker.conf /etc/nginx/conf.d/tracker.conf
# заменить tracker.example.com и пути к сертификатам на свои
sudo nginx -t && sudo systemctl reload nginx
```

* nginx < 1.25.1: удалите строку `http2 on;` и пропишите `listen 443 ssl http2;`.
* Если на сервере нет IPv6 — удалите строки `listen [::]:...`.
* Если нужно отдать трекер по пути (`site.ru/tracker/`) внутри сервера SEO-системы, а не на поддомене,
  перенесите `auth_basic*` в `location /tracker/ { alias /var/www/tracker/; }` — пути в
  `index.html` и `manifest.json` относительные, правки в них не нужны.

Проверка:

```bash
curl -I https://tracker.example.com/                    # 401
curl -I -u myuser:пароль https://tracker.example.com/    # 200
```

## 4. Установка на iPhone

Safari → открыть сайт → ввести логин/пароль → «Поделиться» → «На экран „Домой“».
Приложение откроется без интерфейса Safari.

## Важно знать

* **Хранилище раздельное.** У приложения на экране «Домой» свой `localStorage`, не тот, что во вкладке
  Safari. Записи, сделанные в браузере до установки, в приложении не появятся — перенесите их через
  «Экспорт / Импорт JSON».
* **Делайте экспорт иногда.** Данные живут только на устройстве; при удалении иконки они удаляются.
* **Basic Auth в standalone-режиме.** iOS может снова запросить пароль после того, как система выгрузит
  приложение из памяти. Это нормальное поведение для Basic Auth; связка ключей iCloud подставляет пароль.
* Service Worker не используется (по условию — максимум два файла), поэтому без сети приложение
  не откроется. Для полноценного офлайна понадобится третий файл `sw.js` (~1 КБ).
