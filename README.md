# WordPress и мониторинг ОС

Проект разворачивает WordPress, MariaDB, Nginx, Prometheus, Grafana и node_exporter через Docker Compose. Внешние порты открывает только Nginx: остальные сервисы доступны лишь во внутренних Docker-сетях.

## Требования

- Debian или Ubuntu с установленными Docker Engine и Docker Compose plugin;
- `openssl` для создания самоподписанного сертификата;
- DNS-записи или строки в `/etc/hosts` для `site.local` и `metrics.local`.

## Быстрый запуск

```bash
cp .env.example .env
chmod 600 .env
mkdir -p nginx/certs
openssl req -x509 -nodes -newkey rsa:2048 -sha256 -days 365 \
  -keyout nginx/certs/site.local.key \
  -out nginx/certs/site.local.crt \
  -subj '/CN=site.local' \
  -addext 'subjectAltName=DNS:site.local'
docker compose config --quiet
docker compose up -d
```

Перед запуском необходимо заменить пароли в `.env`. Файл не попадает в Git.

Для локальной проверки добавьте на клиентской машине строку в `/etc/hosts`:

```text
127.0.0.1 site.local metrics.local
```

После запуска откройте `https://site.local` и завершите первоначальную настройку WordPress. Браузер покажет предупреждение о самоподписанном сертификате, это ожидаемо для тестового окружения. Grafana доступна по адресу `http://metrics.local`; учётные данные задаются в `.env`.

## Состав и принятые решения

- `nginx/` содержит конфигурацию фронтенда. Запросы к `site.local` по HTTP перенаправляются на HTTPS и далее передаются в WordPress. `metrics.local` по HTTP передаётся в Grafana.
- Конфигурации Nginx, Prometheus и Grafana расположены на хосте и монтируются в контейнеры в режиме `read-only`.
- WordPress и MariaDB подключены только к закрытой сети `backend`. Prometheus, Grafana и node_exporter находятся в закрытой сети `monitoring`. Nginx подключён к сетям, необходимым для проксирования.
- Данные MariaDB, WordPress, Prometheus и Grafana хранятся в именованных Docker volumes, поэтому не теряются при пересоздании контейнеров.
- В `nginx/conf.d/site.conf` доступ к `/wp-admin` и `/wp-login.php` ограничен. Перед развёртыванием следует заменить примерные приватные сети в директивах `allow` на реальные адреса офиса или VPN.
- Prometheus опрашивает node_exporter раз в 15 секунд. Node_exporter получает метрики ОС через read-only монтирование файловой системы хоста.
- Grafana автоматически получает источник данных Prometheus и дашборд `OS General`. В нём есть входящий и исходящий сетевой трафик, свободное место на `/` и число выделенных/максимальных файловых дескрипторов.
- `fail2ban/jail.local` содержит jail для SSH: 5 неуспешных попыток за 10 минут приводят к блокировке на 1 час. Fail2ban устанавливается на хост, а не в контейнер, чтобы блокировка применялась к SSH хоста.

## Развёртывание Ansible

Скопируйте `ansible/hosts.ini.example` в `ansible/hosts.ini`, укажите IP-адрес сервера и SSH-пользователя, затем выполните на управляющей машине:

```bash
ansible-playbook -i ansible/hosts.ini ansible/deploy.yml
```

Playbook устанавливает Docker, Compose plugin, fail2ban и openssl, копирует необходимые файлы в `/opt/wordpress-monitoring`, создаёт `.env` из шаблона при первом запуске, генерирует сертификат, проверяет Compose-конфигурацию и запускает сервисы. Пароли в `/opt/wordpress-monitoring/.env` нужно изменить до открытия сервера во внешнюю сеть.

## Проверка после запуска

```bash
docker compose ps
docker compose logs --tail=100 nginx prometheus grafana
curl -k --resolve site.local:443:127.0.0.1 https://site.local/
curl --resolve metrics.local:80:127.0.0.1 http://metrics.local/login
```

В Prometheus статус цели `node-exporter` должен быть `UP`. В Grafana в папке `OS` должен появиться дашборд `OS General`.

## CI

GitHub Actions запускается на каждом pull request и push в ветку `main`. Workflow создаёт тестовые переменные и сертификат, выполняет `docker compose config`, поднимает стек и проверяет оба виртуальных хоста Nginx.

## Оценка трудозатрат

Около 6-8 часов: подготовка Compose-конфигурации и Nginx, настройка мониторинга и дашборда, базовая защита, Ansible, CI и проверка.
