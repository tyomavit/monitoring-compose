# Памятка B. С нуля: Prometheus + Grafana в Docker Compose за nginx и HTTPS (grafana.artemdevops.ru)

Задание: на чистом VPS поднять стек для **метрик**: Prometheus (собирает числа с серверов), Grafana (графики и вход по логину), node-exporter (отдаёт метрики этого сервера), Filebeat (отправляет журнал этого сервера в Logstash на сервере A). Grafana открывается из интернета по HTTPS. Prometheus забирает метрики с обоих серверов (A и B), логи B уходят на A напрямую в порт 5044, без SSH-туннелей.

Памятка самодостаточна: ссылок на другие файлы нет. Всё набираем руками, файлы создаём в vim.

Если результат на шаге не совпал с ожидаемым, остановитесь и напишите в чат: номер шага, команду и вывод целиком.

## Паспорт стенда

| | VPS A (памятка A) | VPS B (эта памятка) |
|---|---|---|
| Роль | логи: ELK | метрики: Prometheus + Grafana |
| Публичный IP | `195.208.61.11` | `168.222.140.216` |
| Домен | `kibana.artemdevops.ru` | `grafana.artemdevops.ru` |
| Имя сервера (hostname) | `elk-vps` | `monitoring-vps` |
| Папка стека | `/opt/monitoring-compose/elk` | `/opt/monitoring-compose/prometheus-grafana` |

Порядок: шаги 1-7 можно делать параллельно с A. Запуск (шаг 8) и сквозная проверка (шаг 11) лучше после того, как на A запущены Logstash и node-exporter и открыты порты в ufw.

---

## Шаг 0. Как работаем: терминал, vim, Tab

Работаем в обычном терминале (`ssh root@IP`, например, в терминале VS Code), а не через Remote-SSH: Remote-SSH ставит на сервер «VS Code Server», он съедает сотни мегабайт, а на малом VPS B памяти немного.

**Минимум vim** (vim ставим на шаге 1):

| Что сделать | Клавиши |
|---|---|
| Открыть/создать файл | `vim имя_файла` |
| Режим вставки текста | `i` |
| Выйти из режима вставки | `Esc` |
| Сохранить и выйти | `Esc`, `:wq`, Enter |
| Выйти без сохранения | `Esc`, `:q!`, Enter |
| Удалить строку | `Esc`, `dd` |
| Отменить | `Esc`, `u` |
| Найти текст | `Esc`, `/слово`, Enter |

Настройка под YAML (в YAML только пробелы, табов быть не должно). После установки vim, один раз:

```bash
# пробелы вместо табов, отступ 2 пробела, автоотступ, номера строк
printf 'set expandtab\nset tabstop=2\nset shiftwidth=2\nset softtabstop=2\nset autoindent\nset number\n' >> ~/.vimrc
```

С автоотступом новая строка начинается с отступа предыдущей. Чтобы уменьшить отступ, Backspace (стирает 2 пробела) или `Ctrl+D` в режиме вставки. Если вставляете длинный текст из буфера, сначала `:set paste`.

**Tab в bash** дополняет команды, пути и имена файлов. Двойной Tab показывает варианты. В vim он ничего не дополняет. Стрелка вверх и `Ctrl+R` ускоряют повтор команд.

---

## Шаг 1. Подготовка VPS

### 1.1 Переустановка

1. В панели хостинга: переустановить ОС на Ubuntu 26.04, при переустановке добавить свой SSH-ключ.
2. На ноутбуке:

```powershell
# удаляем из known_hosts старую запись про этот IP
ssh-keygen -R 168.222.140.216
# входим на чистый сервер (на вопрос про отпечаток отвечаем yes)
ssh root@168.222.140.216
```

### 1.2 Система, имя, vim

```bash
# обновляем список пакетов и пакеты
apt update && apt upgrade -y
# ставим vim, git, curl
apt install -y vim git curl
# задаём имя по роли; оно попадёт в host.name записей журнала
hostnamectl set-hostname monitoring-vps
# проверяем
hostnamectl
# если обновилось ядро, перезагружаемся и снова входим
reboot
```

После входа выполните команду про `~/.vimrc` из шага 0.

### 1.3 Вход только по ключу

Убедитесь, что вошли без пароля, по ключу.

```bash
# какие файлы настроек SSH уже есть
ls /etc/ssh/sshd_config.d/
# свой файл; имя с 01, чтобы читаться первым (при повторе параметра побеждает первое значение)
vim /etc/ssh/sshd_config.d/01-hardening.conf
```

Содержимое:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
```

```bash
# проверяем синтаксис (пустой вывод = хорошо)
sshd -t
# смотрим итоговые значения, которые увидит sshd
sshd -T | grep -E 'passwordauthentication|permitrootlogin'
# применяем (текущая сессия не закроется)
systemctl reload ssh
```

Не закрывая текущую сессию, откройте вторую с ноутбука и проверьте вход по ключу. Только потом закрывайте старую.

---

## Шаг 2. DNS

Если вы уже создали записи по памятке A, достаточно проверки. Иначе в панели DNS домена `artemdevops.ru` создайте A-записи (TTL 300):

| Имя | Тип | Значение |
|---|---|---|
| `kibana` | A | `195.208.61.11` |
| `grafana` | A | `168.222.140.216` |

```bash
# какой IP у имени; должен быть 168.222.140.216 (если пусто, подождите 5-10 минут)
getent hosts grafana.artemdevops.ru
```

Почему `getent`, а не `dig`/`nslookup`: на чистой Ubuntu они не установлены (пакет `bind9-dnsutils`), а `getent` есть всегда и спрашивает так же, как сами программы на сервере (через системный резолвер). Если `dig` стоит, можно `dig +short grafana.artemdevops.ru`.

---

## Шаг 3. Swap

```bash
# проверяем память и swap
free -h
# создаём файл подкачки 2 ГБ
fallocate -l 2G /swapfile
# закрываем файл от чужих
chmod 600 /swapfile
# размечаем под swap
mkswap /swapfile
# включаем
swapon /swapfile
# чтобы swap включался после перезагрузки
echo '/swapfile none swap sw 0 0' >> /etc/fstab
# проверяем
free -h
```

---

## Шаг 4. Docker

```bash
# проверяем доступность реестров (401 нормален) и репозитория Docker (ожидаем 200)
curl -sI https://docker.elastic.co/v2/ | head -n 1
curl -sI https://registry-1.docker.io/v2/ | head -n 1
curl -sI https://download.docker.com/linux/ubuntu/gpg | head -n 1
# подготовка к подключению репозитория
apt update
apt install -y ca-certificates curl
# папка для ключей и ключ репозитория Docker
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
# подключаем репозиторий Docker (длинная команда, можно скопировать)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" > /etc/apt/sources.list.d/docker.list
# ставим Docker и плагин Compose
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# проверяем версии
docker --version
docker compose version
# (по желанию) проверяем, что образы скачиваются: блокировки бывают неравномерными. Скачанное лежит в кэше сервера, `docker compose up` возьмёт его оттуда. Шаг можно пропустить: `up` скачает всё сам, а проверка нужна, чтобы заранее увидеть блокировку
docker pull prom/prometheus:v3.13.2
docker pull docker.elastic.co/beats/filebeat:9.5.5
```

Если `apt update` ругается на репозиторий Docker для Ubuntu 26.04, пришлите вывод в чат.

---

## Шаг 5. Файрвол

Помните: порты, **опубликованные Docker** (`ports:`), ufw обходит, поэтому Prometheus и Grafana публикуются только на `127.0.0.1`. А node-exporter работает в режиме `host`, и ходит к нему контейнер Prometheus из подсети Docker этого стека. Для этого нужно отдельное правило.

```bash
# SSH первым
ufw allow 22/tcp
# HTTP и HTTPS: nginx и сертификат
ufw allow 80/tcp
ufw allow 443/tcp
# контейнер Prometheus (подсеть 172.30.0.0/24) ходит на порт 9100 хоста
ufw allow proto tcp from 172.30.0.0/24 to any port 9100
# включаем (отвечаем y)
ufw enable
# итог
ufw status verbose
```

Исходящие соединения (к A на 5044 и 9100) ufw не блокирует: по умолчанию исходящее разрешено.

Прогноз: если Prometheus (в контейнере) не получит ответ от node-exporter, какое состояние цели он покажет и где вы это увидите? (Проверим в шаге 8.)

---

## Шаг 6. Репозиторий

### 6.1 На GitHub

Репозиторий `monitoring-compose` уже создан по памятке A (Public, README, шаблон `.gitignore`: None). Если нет: создайте так же.

### 6.2 Клонирование

```bash
# общая папка проектов
mkdir -p /opt
# переходим
cd /opt
# клонируем (подставьте логин)
git clone https://github.com/<ваш-логин>/monitoring-compose.git
# заходим
cd monitoring-compose
# содержимое со скрытыми файлами
ls -la
```

### 6.3 Первым файлом .gitignore, затем структура

Если `.gitignore` уже есть (создан на A и отправлен), проверьте через `cat .gitignore`. Если нет, создайте:

```bash
vim .gitignore
```

```
.env
*.key
data/
prometheus-grafana/prometheus/targets/node.yml
```

(Последняя строка нужна, если вы решите хранить реальный список серверов вне репозитория; шаблон-пример можно положить в `node.yml.example`.)

```bash
# создаём папки стека
mkdir -p prometheus-grafana/prometheus/targets prometheus-grafana/grafana/provisioning/datasources prometheus-grafana/filebeat prometheus-grafana/nginx
# переходим в папку стека
cd prometheus-grafana
# проверяем структуру
find . -type d
```

Вся работа дальше в `/opt/monitoring-compose/prometheus-grafana`.

---

## Шаг 7. Конфигурационные файлы

### 7.1 Секреты: `.env`

```bash
# пароль админа Grafana: случайная строка из букв и цифр
openssl rand -hex 16
```

Сохраните значение в менеджер паролей.

```bash
vim .env
```

Содержимое (вместо `...` подставьте сгенерированный пароль; без кавычек и пробелов):

```
PROMETHEUS_VERSION=v3.13.2
GRAFANA_VERSION=13.1.3
NODE_EXPORTER_VERSION=v1.12.1
ELASTIC_VERSION=9.5.5
GRAFANA_DOMAIN=grafana.artemdevops.ru
GRAFANA_ADMIN_PASSWORD=...
HOST_NAME=monitoring-vps
LOGSTASH_HOST=195.208.61.11
```

```bash
# закрываем от других пользователей
chmod 600 .env
# git должен игнорировать этот файл (в ответе будет строка с правилом)
git check-ignore -v .env
```

### 7.1б Шаблон `.env.example` (для репозитория)

Реальный `.env` в репозиторий не попадает, поэтому рядом кладём шаблон: версии и домен остаются, секрет и IP сервера A заменены заглушками. При развёртывании из репозитория достаточно скопировать шаблон в `.env` и проставить значения.

```bash
# строим шаблон из рабочего .env: пароль -> CHANGE_ME, IP сервера A -> IP_OF_ELK_VPS (-E включает расширенные регулярные выражения, \1 возвращает имя переменной вместе с =)
sed -E -e 's/^(GRAFANA_ADMIN_PASSWORD=).*/\1CHANGE_ME/' -e 's/^(LOGSTASH_HOST=).*/\1IP_OF_ELK_VPS/' .env > .env.example
# проверяем глазами: секретов в файле быть не должно, версии на месте
cat .env.example
# убеждаемся, что git НЕ игнорирует шаблон (вывод пустой = не игнорируется)
git check-ignore -v .env.example
```

Развёртывание из репозитория потом: `cp .env.example .env`, `vim .env` (вписать пароль через `openssl rand -hex 16` и IP сервера A), `chmod 600 .env`.

### 7.2 Конфиг Prometheus

```bash
vim prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'monitoring-vps'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    file_sd_configs:
      - files:
          - /etc/prometheus/targets/node.yml
```

Что здесь что:
- `scrape_interval: 15s`: Prometheus опрашивает цели каждые 15 секунд. Это модель **pull**: он сам приходит за метриками (в отличие от логов, которые Filebeat сам **отправляет**).
- Задание `prometheus` следит за самим собой.
- Задание `node` берёт список серверов из отдельного файла (`file_sd_configs`): список можно править без перезапуска Prometheus.

### 7.3 Список серверов

```bash
vim prometheus/targets/node.yml
```

```yaml
- targets: ['172.30.0.1:9100']
  labels:
    host: 'monitoring-vps'

- targets: ['195.208.61.11:9100']
  labels:
    host: 'elk-vps'
```

`172.30.0.1` это адрес шлюза подсети Docker этого стека (то есть самого сервера, видимый из контейнера); именно там node-exporter слушает на этом сервере. Метка `host` понадобится для фильтров в Grafana.

### 7.4 Источник данных Grafana (файлом, при старте)

```bash
vim grafana/provisioning/datasources/prometheus.yml
```

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

Что здесь что: Grafana сама подключит Prometheus при старте, вручную настраивать не придётся. `http://prometheus:9090`: контейнеры одной сети Compose находят друг друга по имени сервиса.

### 7.5 Конфиг Filebeat

```bash
vim filebeat/filebeat.yml
```

```yaml
filebeat.inputs:
  - type: journald
    id: host-journald
    seek: tail

output.logstash:
  hosts: ["${LOGSTASH_HOST}:5044"]
```

`${LOGSTASH_HOST}` Filebeat берёт из переменной окружения контейнера (задаётся в compose из `.env`). `seek: tail`: при первом запуске только новые записи.

### 7.6 docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
name: monitoring

services:
  prometheus:
    image: prom/prometheus:${PROMETHEUS_VERSION}
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus:/etc/prometheus:ro
      - prometheus_data:/prometheus
    ports:
      - "127.0.0.1:9090:9090"

  grafana:
    image: grafana/grafana:${GRAFANA_VERSION}
    restart: unless-stopped
    depends_on:
      - prometheus
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_DOMAIN=${GRAFANA_DOMAIN}
      - GF_SERVER_ROOT_URL=https://${GRAFANA_DOMAIN}/
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - grafana_data:/var/lib/grafana
    ports:
      - "127.0.0.1:3000:3000"

  node-exporter:
    image: prom/node-exporter:${NODE_EXPORTER_VERSION}
    restart: unless-stopped
    pid: host
    network_mode: host
    command:
      - '--path.rootfs=/host'
      - '--web.listen-address=172.30.0.1:9100'
      - '--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($$|/)'
    volumes:
      - '/:/host:ro,rslave'

  filebeat:
    image: docker.elastic.co/beats/filebeat:${ELASTIC_VERSION}
    restart: unless-stopped
    hostname: ${HOST_NAME}
    user: root
    command: ["-e", "--strict.perms=false"]
    environment:
      - LOGSTASH_HOST=${LOGSTASH_HOST}
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - filebeat_data:/usr/share/filebeat/data
      - /var/log/journal:/var/log/journal:ro
      - /run/log/journal:/run/log/journal:ro
      - /etc/machine-id:/etc/machine-id:ro

networks:
  default:
    ipam:
      config:
        - subnet: 172.30.0.0/24

volumes:
  prometheus_data:
  grafana_data:
  filebeat_data:
```

Что здесь что:

| Строка | Смысл |
|---|---|
| `name: monitoring` | имя проекта (сеть будет `monitoring_default`) |
| `ports: "127.0.0.1:9090:9090"` | дверь наружу только внутри сервера; снаружи войдёт nginx |
| `./prometheus:/etc/prometheus:ro` | конфиги монтируются «только для чтения» |
| `prometheus_data`, `grafana_data` | именованные тома: данные переживают пересоздание контейнера |
| `--storage.tsdb.retention.time=15d` | метрики хранятся 15 дней |
| `--web.enable-lifecycle` | позволяет перечитывать конфиг командой `curl -X POST .../-/reload` |
| `GF_USERS_ALLOW_SIGN_UP=false` | в Grafana нельзя зарегистрироваться самим |
| `GF_SERVER_ROOT_URL` | внешний адрес Grafana (за nginx) |
| `pid: host`, `network_mode: host` у node-exporter | должен видеть сам сервер, а не «комнату контейнера» |
| `--web.listen-address=172.30.0.1:9100` | слушает адрес шлюза подсети: туда достаёт Prometheus из контейнера, а из интернета этот порт недоступен |
| `subnet: 172.30.0.0/24` | задаём подсеть сами, чтобы адрес шлюза `172.30.0.1` был известен заранее |
| `hostname: ${HOST_NAME}` у Filebeat | иначе поле `host.name` будет случайным ID контейнера |
| `$$` в маске | Compose превращает в `$` |

```bash
# проверка структуры файла
docker compose config --quiet
# проверка подставленных значений: версии, порты и адрес шлюза
docker compose config | grep -E 'image:|127.0.0.1|172.30|hostname|LOGSTASH_HOST'
```

---

## Шаг 8. Запуск и первые проверки

```bash
# проверим, доступен ли Logstash сервера A (open = порт принимает соединения; если A ещё не готов, вернёмся)
timeout 5 bash -c '</dev/tcp/195.208.61.11/5044' && echo open || echo closed
# запускаем все четыре сервиса
docker compose up -d
# все Up?
docker compose ps
# кто слушает порты (ожидаем 127.0.0.1:9090, 127.0.0.1:3000, 172.30.0.1:9100)
ss -tlnp | grep -E ':(9090|3000|9100)\b'
```

Если `node-exporter` не стартует (адрес `172.30.0.1` появляется только вместе с сетью): `docker compose logs --tail 40 node-exporter`, потом повторите `docker compose up -d`.

```bash
# цели Prometheus: адрес и состояние; ожидаем health "up" у обеих
curl -s http://127.0.0.1:9090/api/v1/targets | grep -o '"scrapeUrl":"[^"]*"\|"health":"[^"]*"'
```

Если `172.30.0.1:9100` в состоянии `down`: проверьте правило ufw (шаг 5) и адрес шлюза:

```bash
# адрес шлюза сети проекта
docker network inspect monitoring_default --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'
```

Если цель `195.208.61.11:9100` `down`: на A нет правила ufw для IP `168.222.140.216` порта 9100 или не запущен node-exporter.

Запасной вариант, если схема со шлюзом не заработает за 45 минут: напишите в чат, переведём Prometheus, Grafana и node-exporter на `network_mode: host`.

Скорость подхвата списка серверов: добавьте во второй файл `targets/node.yml` ещё одну строку-цель (любой выдуманный адрес), засеките время до появления в `/api/v1/targets` и верните как было. Запишите результат.

---

## Шаг 9. nginx и HTTPS

```bash
# nginx и certbot (программа получает сертификаты Let's Encrypt) с плагином для nginx
apt install -y nginx certbot python3-certbot-nginx
# убираем сайт-заглушку по умолчанию
rm -f /etc/nginx/sites-enabled/default
# конфиг сайта
vim /etc/nginx/sites-available/grafana
```

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name grafana.artemdevops.ru;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }
}
```

Что здесь что: `server_name` выбирает домен; `proxy_pass` передаёт запрос Grafana; заголовки `Host`, `X-Forwarded-*` сообщают Grafana внешние данные запроса; `Upgrade`/`Connection` нужны для WebSocket (Grafana Live).

```bash
# включаем сайт ссылкой
ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/grafana
# проверяем конфиг
nginx -t
# перечитываем без остановки
systemctl reload nginx
# проверка с самого сервера (ожидаем 302)
curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: grafana.artemdevops.ru' http://127.0.0.1/
# выпускаем сертификат, certbot допишет HTTPS и редирект (подставьте почту)
certbot --nginx -d grafana.artemdevops.ru --non-interactive --agree-tos -m <ваша-почта> --redirect
# пробное автопродление
certbot renew --dry-run
# что certbot дописал
cat /etc/nginx/sites-available/grafana
# копия рабочего конфига в репозиторий
cp /etc/nginx/sites-available/grafana /opt/monitoring-compose/prometheus-grafana/nginx/grafana.conf
```

Не повторяйте выпуск сертификата много раз подряд при ошибках (лимит неудачных попыток). Сначала найдите причину: DNS, порт 80, имя в `server_name`.

Откройте `https://grafana.artemdevops.ru`: форма входа. Логин `admin`, пароль из `.env`.

---

## Шаг 10. Дашборд и пользователь-зритель

### 10.1 Дашборд

В Grafana: Dashboards → New → Import → номер `1860` (Node Exporter Full) → выбрать источник Prometheus → Import. В списке Nodename должны быть оба сервера. Если Import не может скачать дашборд (блокировка grafana.com), напишите в чат.

### 10.2 Пользователь для друга

```bash
# вводим пароли без показа
read -rsp 'Пароль для друга: ' FRIEND_PWD; echo
read -rsp 'Пароль admin: ' ADMIN_PWD; echo
# создаём пользователя через API Grafana (роль по умолчанию Viewer: только просмотр)
curl -s -u "admin:$ADMIN_PWD" -X POST http://127.0.0.1:3000/api/admin/users -H 'Content-Type: application/json' -d "{\"name\":\"Andrey\",\"login\":\"andrey\",\"email\":\"andrey@example.com\",\"password\":\"$FRIEND_PWD\"}"
# стираем переменные
unset FRIEND_PWD ADMIN_PWD
```

Проверка: войдите как `andrey`; дашборд виден, настройки недоступны. В Administration → Users роль должна быть Viewer (если нет, смените вручную).

---

## Шаг 11. Сквозная проверка

```bash
# Filebeat без ошибок соединения с 5044?
docker compose logs --tail 40 filebeat | tail -n 15
# создаём тестовую запись в журнал
logger "тест с monitoring-vps"
# обе цели Prometheus up?
curl -s http://127.0.0.1:9090/api/v1/targets | grep -o '"scrapeUrl":"[^"]*"\|"health":"[^"]*"'
```

В Kibana (`https://kibana.artemdevops.ru`, Discover, `filebeat-*`): найдите `тест с monitoring-vps`, в поле `host.name` значение `monitoring-vps`. Это и есть главное: логи B дошли до A.

---

## Шаг 12. Перезагрузка

```bash
# перезагружаем сервер
reboot
```

После входа: `cd /opt/monitoring-compose/prometheus-grafana && docker compose ps`: все Up без вашего вмешательства (особенно node-exporter: ему нужна сеть со шлюзом 172.30.0.1), цели Prometheus up, сайт открывается, новая запись из `logger` доходит до Kibana.

---

## Шаг 13. Фиксация в репозитории (после всех проверок)

```bash
cd /opt/monitoring-compose
# имя и почта для коммитов (только для этого репозитория)
git config user.name "Artyom"
git config user.email "<почта GitHub>"
# что изменилось: .env быть не должно, а .env.example должен быть в списке
git status
# добавляем и проверяем ещё раз
git add .
git status
# коммит
git commit -m "Prometheus + Grafana: compose, конфиги, nginx"
```

Доступ на запись: deploy key.

```bash
# ключ без пароля
ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -N "" -C "monitoring-vps"
# показываем открытую часть: вставьте на GitHub (репозиторий → Settings → Deploy keys → Add, Allow write access)
cat ~/.ssh/github_deploy.pub
# SSH использует этот ключ для github.com
printf 'Host github.com\n  IdentityFile ~/.ssh/github_deploy\n  IdentitiesOnly yes\n' >> ~/.ssh/config
# переключаем адрес репозитория на SSH
git remote set-url origin git@github.com:<ваш-логин>/monitoring-compose.git
# проверка (successfully authenticated)
ssh -T git@github.com
# забираем изменения с A (если он уже отправил) и кладём наши поверх
git pull --rebase origin main
# отправляем
git push origin main
```

Если Git скажет про конфликт, пришлите вывод в чат: разберём.

---

## Что записывать для финальной памятки

- Заработала ли схема со шлюзом `172.30.0.1` + правило ufw с первого раза; если нет, что именно было.
- Через сколько секунд Prometheus подхватил изменение `targets/node.yml`.
- Прошёл ли `certbot` с первой попытки.
- Поднялся ли node-exporter после перезагрузки.
- Открылся ли дашборд 1860.
- Свободная память в покое (`free -h`).
- Где вы спотыкались с Tab/vim.

## Типичные ошибки

| Симптом | Причина и действие |
|---|---|
| Цель `172.30.0.1:9100` down | нет правила ufw (шаг 5) или node-exporter не стартовал |
| Цель `195.208.61.11:9100` down | на A нет правила ufw для IP B или контейнер не запущен |
| Filebeat: connection refused / timeout до 5044 | на A не запущен Logstash или нет правила ufw для IP B |
| Grafana: Bad Gateway | контейнер ещё стартует: `docker compose logs --tail 40 grafana` |
| Пустой список целей | нет файла `prometheus/targets/node.yml` или опечатка в нём (смотрите `docker compose logs prometheus`) |
| `host.name` в Kibana вида `abc123def` | не задан `hostname: ${HOST_NAME}` у Filebeat |
| Дашборд не видит данные | неверный источник данных (Prometheus) или метки `host` |

## Известные ограничения

- Логи B идут к A и метрики A идут к B по интернету открытым текстом, защита только файрвол по IP.
- Нет Alertmanager (алертов).
- Схема со шлюзом `172.30.0.1` не проверялась заранее, запасной вариант: режим `host`.
