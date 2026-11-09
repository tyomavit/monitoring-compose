# Памятка A. С нуля: ELK в Docker Compose за nginx и HTTPS (kibana.artemdevops.ru)

Задание: на чистом VPS поднять стек для **логов**: Elasticsearch (хранилище), Logstash (приёмник и обработчик), Kibana (веб-интерфейс), Filebeat (отправляет журнал самого сервера), node-exporter (отдаёт метрики серверу B). Kibana открывается из интернета по HTTPS с логином. Логи сервера B приходят сюда напрямую на порт 5044, без SSH-туннелей.

Памятка полностью самодостаточна: ссылок на другие файлы нет. Всё набираем руками, vim вместо редакторов VS Code.

Если на каком-то шаге результат не совпал с ожидаемым, остановитесь и напишите в чат: номер шага, команду и вывод целиком.

## Паспорт стенда (значения для подстановки)

| | VPS A (эта памятка) | VPS B (памятка B) |
|---|---|---|
| Роль | логи: ELK | метрики: Prometheus + Grafana |
| Публичный IP | `195.208.61.11` | `168.222.140.216` |
| Домен | `kibana.artemdevops.ru` | `grafana.artemdevops.ru` |
| Имя сервера (hostname) | `elk-vps` | `monitoring-vps` |
| Папка стека | `/opt/monitoring-compose/elk` | `/opt/monitoring-compose/prometheus-grafana` |

Названия в панели хостинга (`kibana`, `grafana`) это просто ярлыки. В работе hostname называют по роли, а ярлык в панели может быть любым.

Про слово «мониторинг»: это общий термин, и ELK, и Prometheus его части. Принято так: **метрики** (числа во времени: процессор, память, диск) это Prometheus + Grafana; **логи** (события: что произошло и когда) это ELK. Вместе с трассировками это называют наблюдаемостью (observability).

---

## Шаг 0. Как работаем: терминал, vim, Tab

Работаем в обычном терминале (например, в терминале VS Code: `ssh root@IP`), а не через Remote-SSH. Причина: Remote-SSH ставит на сервер свой «VS Code Server», он занимает сотни мегабайт памяти, а на малых VPS память нужна Elasticsearch.

**Минимум vim** (после установки vim, шаг 1):

| Что сделать | Клавиши |
|---|---|
| Открыть/создать файл | `vim имя_файла` |
| Войти в режим вставки текста | `i` |
| Выйти из режима вставки | `Esc` |
| Сохранить и выйти | `Esc`, затем `:wq`, Enter |
| Выйти без сохранения | `Esc`, затем `:q!`, Enter |
| Удалить строку | `Esc`, затем `dd` |
| Отменить | `Esc`, затем `u` |
| Найти текст | `Esc`, затем `/слово`, Enter |
| Номера строк | `:set number` |

Настройка vim под YAML (файлы YAML требуют пробелов, а не табуляции). Выполняется один раз после установки vim:

```bash
# создаём/дописываем настройки vim: пробелы вместо табов, отступ 2 пробела, автоотступ
printf 'set expandtab\nset tabstop=2\nset shiftwidth=2\nset softtabstop=2\nset autoindent\nset number\n' >> ~/.vimrc
```

Практика с автоотступом: после строки с отступом следующая начинается с того же отступа. Чтобы уменьшить отступ, нажмите Backspace (стирает 2 пробела) или `Ctrl+D` в режиме вставки. Если вставляете длинный текст из буфера, перед этим в vim наберите `:set paste`, иначе отступы поедут.

**Tab в bash** дополняет команды, имена файлов и папок (`cd /opt/mon<Tab>`). Двойное нажатие показывает варианты. В vim и в тексте файла Tab ничего не дополняет. Для `docker compose` дополнение работает частично; для имён сервисов не рассчитывайте на него. Стрелка вверх и `Ctrl+R` (поиск по истории) ускоряют повтор команд.

---

## Шаг 1. Подготовка VPS

### 1.1 Переустановка

1. В панели хостинга: переустановить ОС на Ubuntu 26.04, при переустановке выбрать/добавить свой SSH-ключ.
2. На ноутбуке удалить старый отпечаток сервера (иначе SSH откажется подключаться):

```powershell
# удаляем из known_hosts запись про этот IP: после переустановки у сервера новый ключ
ssh-keygen -R 195.208.61.11
# входим на чистый сервер (на вопрос про отпечаток отвечаем yes)
ssh root@195.208.61.11
```

### 1.2 Система, имя, vim

```bash
# обновляем список пакетов и устанавливаем обновления
apt update && apt upgrade -y
# ставим vim и git (в минимальной Ubuntu их может не быть) и curl
apt install -y vim git curl
# задаём имя сервера по роли; оно попадёт в поле host.name каждой записи журнала
hostnamectl set-hostname elk-vps
# проверяем
hostnamectl
# если ядро обновилось, перезагружаемся и снова входим
reboot
```

После входа выполните команду из шага 0 про `~/.vimrc`.

### 1.3 Вход только по ключу

Проверьте, что вход по ключу работает (вы вошли без пароля). Затем:

```bash
# смотрим, какие файлы настроек SSH уже есть
ls /etc/ssh/sshd_config.d/
# создаём свой файл; имя начинается с 01, чтобы читаться первым (при повторе параметра побеждает первое значение)
vim /etc/ssh/sshd_config.d/01-hardening.conf
```

Содержимое файла:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
```

```bash
# проверяем синтаксис (пустой вывод = всё хорошо)
sshd -t
# смотрим итоговое значение, которое увидит sshd
sshd -T | grep -E 'passwordauthentication|permitrootlogin'
# применяем (текущая сессия не закроется)
systemctl reload ssh
```

Не закрывая текущую сессию, откройте **вторую** с ноутбука и убедитесь, что вход по ключу работает. Только после этого закрывайте старую.

---

## Шаг 2. DNS

В панели DNS домена `artemdevops.ru` создайте две A-записи с TTL 300 (5 минут; при ошибке быстро исправите):

| Имя | Тип | Значение |
|---|---|---|
| `kibana` | A | `195.208.61.11` |
| `grafana` | A | `168.222.140.216` |

```bash
# спрашиваем, какой IP у имени; должен выдать 195.208.61.11 (если пусто, подождите 5-10 минут)
getent hosts kibana.artemdevops.ru
```

Без работающего DNS шаг 9 (сертификат) не получится. Запишите, через сколько минут имя стало отвечать.

---

## Шаг 3. Swap и параметры ядра

```bash
# проверяем, сколько памяти и есть ли swap (для ELK желательно от 4 ГБ)
free -h
# создаём файл подкачки 2 ГБ: запас на случай всплесков памяти
fallocate -l 2G /swapfile
# закрываем файл от чужих (только владелец читает/пишет), иначе swapon ругнётся
chmod 600 /swapfile
# размечаем файл как область подкачки
mkswap /swapfile
# включаем
swapon /swapfile
# чтобы swap включался после перезагрузки, добавляем строку в /etc/fstab
echo '/swapfile none swap sw 0 0' >> /etc/fstab
# ядро реже сбрасывает память в swap
echo 'vm.swappiness=10' > /etc/sysctl.d/90-swappiness.conf
# Elasticsearch требует не меньше 262144 областей памяти (иначе не стартует)
echo 'vm.max_map_count=262144' > /etc/sysctl.d/90-elasticsearch.conf
# применяем все настройки sysctl сразу
sysctl --system
# проверяем результат
free -h
sysctl vm.swappiness vm.max_map_count
```

Если памяти меньше 4 ГБ, напишите в чат до запуска (надо будет уменьшить размеры памяти у Elasticsearch/Logstash).

---

## Шаг 4. Docker

```bash
# проверяем, доступны ли реестры (401 нормален: сервер отвечает и просит авторизацию)
curl -sI https://docker.elastic.co/v2/ | head -n 1
curl -sI https://registry-1.docker.io/v2/ | head -n 1
# проверяем репозиторий Docker (ожидаем 200)
curl -sI https://download.docker.com/linux/ubuntu/gpg | head -n 1
# подготовка к подключению репозитория
apt update
apt install -y ca-certificates curl
# папка для ключей и ключ репозитория Docker
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
# подключаем репозиторий Docker (кодовое имя Ubuntu подставляется автоматически) - длинная команда, можно скопировать
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" > /etc/apt/sources.list.d/docker.list
# ставим Docker и плагин Compose
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# проверяем версии
docker --version
docker compose version
# проверяем, что нужный образ скачивается (заодно кладём его в кэш)
docker pull docker.elastic.co/elasticsearch/elasticsearch:9.5.5
```

Если `apt update` ругается на репозиторий Docker для Ubuntu 26.04 (нет кодового имени), напишите в чат вывод: найдём обходной вариант.

---

## Шаг 5. Файрвол

Логика: по умолчанию закрыть всё входящее и открыть только нужное.

Ловушка: порты, **опубликованные Docker** (`ports:` в compose), файрвол ufw **обходит**: Docker правит iptables сам. Поэтому в compose мы публикуем порты только на `127.0.0.1`. А Logstash и node-exporter работают в режиме `network_mode: host` (живут прямо в сети сервера), и для них ufw действует.

```bash
# SSH первым, иначе отрежем себе доступ
ufw allow 22/tcp
# HTTP и HTTPS: для nginx и для выпуска сертификата
ufw allow 80/tcp
ufw allow 443/tcp
# Logstash принимает логи только от сервера B
ufw allow from 168.222.140.216 to any port 5044 proto tcp
# node-exporter отдаёт метрики только серверу B
ufw allow from 168.222.140.216 to any port 9100 proto tcp
# включаем (на вопрос отвечаем y; SSH-сессия не оборвётся, 22-й уже разрешён)
ufw enable
# смотрим итог
ufw status verbose
```

Прогноз перед продолжением: что будет, если с ноутбука выполнить `curl http://195.208.61.11:5044`? (Запишите ответ, проверим в шаге 8.)

---

## Шаг 6. Репозиторий

### 6.1 На GitHub (с ноутбука)

Создайте репозиторий `monitoring-compose`: **Public**, галочка **Add a README**, шаблон `.gitignore` выбирайте **None** (его мы напишем сами, шаблоны там под языки программирования, а не под Docker). Публичный репозиторий означает, что пароли в нём недопустимы: поэтому первым файлом будет `.gitignore`.

### 6.2 Клонирование на сервер

```bash
# общая папка для проектов
mkdir -p /opt
# переходим в неё
cd /opt
# клонируем свой репозиторий (подставьте логин)
git clone https://github.com/<ваш-логин>/monitoring-compose.git
# заходим внутрь
cd monitoring-compose
# смотрим содержимое, включая скрытые файлы
ls -la
```

### 6.3 Первый файл: .gitignore, затем структура

```bash
# создаём файл
vim .gitignore
```

Содержимое:

```
.env
*.key
data/
```

```bash
# создаём папки стека одной командой: elk и вложенные logstash/pipeline, filebeat, nginx
mkdir -p elk/logstash/pipeline elk/filebeat elk/nginx
# переходим в папку стека
cd elk
# проверяем структуру
find . -type d
```

Теперь вся работа ведётся в `/opt/monitoring-compose/elk`.

---

## Шаг 7. Конфигурационные файлы

### 7.1 Секреты: `.env`

```bash
# генерируем 4 случайные строки из букв и цифр (hex подходит: Compose не любит $ и #)
openssl rand -hex 16
openssl rand -hex 16
openssl rand -hex 16
openssl rand -hex 32
```

Запишите значения в менеджер паролей (по одному на `ELASTIC_PASSWORD`, `KIBANA_PASSWORD`, `LOGSTASH_INTERNAL_PASSWORD`, и длинное для `KIBANA_ENCRYPTION_KEY`).

```bash
# создаём файл
vim .env
```

Содержимое (вместо `...` подставьте сгенерированные значения, без кавычек и пробелов):

```
ELASTIC_VERSION=9.5.5
NODE_EXPORTER_VERSION=v1.12.1
KIBANA_DOMAIN=kibana.artemdevops.ru
ELASTIC_PASSWORD=...
KIBANA_PASSWORD=...
LOGSTASH_INTERNAL_PASSWORD=...
KIBANA_ENCRYPTION_KEY=...
```

```bash
# закрываем файл от других пользователей
chmod 600 .env
# проверяем, что git его игнорирует: должна быть строка с правилом .gitignore
git check-ignore -v .env
```

Для чего `.env`: Compose сам читает его и подставляет `${ИМЯ}` в `docker-compose.yml`. Версии образов тоже вынесены сюда, чтобы обновление было правкой в одном месте.

### 7.2 Pipeline Logstash

```bash
vim logstash/pipeline/beats-to-es.conf
```

```
input {
  beats {
    port => 5044
  }
}

filter {
  mutate {
    add_tag => ["via_logstash"]
  }
}

output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    user => "logstash_internal"
    password => "${LOGSTASH_INTERNAL_PASSWORD}"
    data_stream => "false"
    index => "%{[@metadata][beat]}-%{[@metadata][version]}"
    action => "create"
    manage_template => false
    ilm_enabled => false
  }
}
```

Что здесь что:
- `input beats port 5044`: Logstash слушает порт, куда Filebeat шлёт события.
- `filter mutate add_tag`: ставит пометку, что запись прошла через Logstash. Здесь будущее место для разбора и чистки полей.
- `output`: Logstash записывает в Elasticsearch под ограниченным пользователем `logstash_internal`. `index` собирает имя data stream из метаданных Filebeat (`filebeat-9.5.5`). `action => create` обязателен для data stream. `manage_template` и `ilm_enabled => false`: шаблон и политику создаёт `filebeat setup`, а не Logstash.
- `127.0.0.1:9200`: Logstash работает в режиме host и доходит до Elasticsearch через порт, который опубликован на loopback.

### 7.3 Конфиг Filebeat

```bash
vim filebeat/filebeat.yml
```

```yaml
filebeat.inputs:
  - type: journald
    id: host-journald
    seek: tail

output.logstash:
  hosts: ["127.0.0.1:5044"]
```

Что здесь что:
- `journald`: читаем системный журнал (`journalctl`) сервера.
- `seek: tail`: при первом запуске берём только новые записи, историю не тянем.
- Filebeat помнит позицию («закладку») в томе `filebeat_data`, поэтому после перезапуска продолжает с места остановки (при сбоях возможны дубли).

### 7.4 docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
name: elk

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${ELASTIC_VERSION}
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - xpack.security.http.ssl.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "127.0.0.1:9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:${ELASTIC_VERSION}
    restart: unless-stopped
    depends_on:
      - elasticsearch
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
      - SERVER_PUBLICBASEURL=https://${KIBANA_DOMAIN}
      - XPACK_SECURITY_ENCRYPTIONKEY=${KIBANA_ENCRYPTION_KEY}
      - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=${KIBANA_ENCRYPTION_KEY}
    ports:
      - "127.0.0.1:5601:5601"

  logstash:
    image: docker.elastic.co/logstash/logstash:${ELASTIC_VERSION}
    restart: unless-stopped
    network_mode: host
    depends_on:
      - elasticsearch
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
      - XPACK_MONITORING_ENABLED=false
      - HTTP_HOST=127.0.0.1
      - LOGSTASH_INTERNAL_PASSWORD=${LOGSTASH_INTERNAL_PASSWORD}
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro

  filebeat:
    image: docker.elastic.co/beats/filebeat:${ELASTIC_VERSION}
    restart: unless-stopped
    network_mode: host
    user: root
    command: ["-e", "--strict.perms=false"]
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - filebeat_data:/usr/share/filebeat/data
      - /var/log/journal:/var/log/journal:ro
      - /run/log/journal:/run/log/journal:ro
      - /etc/machine-id:/etc/machine-id:ro

  node-exporter:
    image: prom/node-exporter:${NODE_EXPORTER_VERSION}
    restart: unless-stopped
    pid: host
    network_mode: host
    command:
      - '--path.rootfs=/host'
      - '--web.listen-address=:9100'
      - '--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($$|/)'
    volumes:
      - '/:/host:ro,rslave'

volumes:
  es_data:
  filebeat_data:
```

Что здесь что:

| Строка | Смысл |
|---|---|
| `name: elk` | имя проекта; от него зависят имена сети и томов, не зависящие от названия папки |
| `image: ...:${ELASTIC_VERSION}` | версия берётся из `.env` |
| `restart: unless-stopped` | контейнер поднимается после сбоя и перезагрузки сервера |
| `discovery.type=single-node` | один узел без кластера |
| `xpack.security.http.ssl.enabled=false` | учебное упрощение: внутри сети Docker без TLS |
| `ES_JAVA_OPTS=-Xms1g -Xmx1g` | память Java: ровно 1 ГБ, начальная = максимальной |
| `ports: "127.0.0.1:9200:9200"` | порт открыт только внутри сервера (помните, Docker обходит ufw) |
| `ELASTICSEARCH_HOSTS=http://elasticsearch:9200` | Kibana находит Elasticsearch по имени сервиса («телефонная книга» сети Compose) |
| `kibana_system` | служебный пользователь, под которым Kibana ходит в Elasticsearch |
| `SERVER_PUBLICBASEURL` | внешний адрес Kibana (за nginx) |
| `network_mode: host` | контейнер живёт в сети сервера: `127.0.0.1` внутри совпадает с сервером, имена сервисов (`elasticsearch`) там **не работают**, зато ufw действует. Нужно Logstash (принимает чужие подключения на 5044), Filebeat (простое попадание на 127.0.0.1:5044) и node-exporter (должен видеть сам сервер) |
| `user: root`, mounts журнала | Filebeat читает системный журнал сервера |
| `--strict.perms=false` | Filebeat не требует строгих прав на смонтированный конфиг |
| `$$` в маске node-exporter | Compose превращает `$$` в одиночный `$` |
| `volumes:` внизу | именованные тома для данных: переживают пересоздание контейнеров |

```bash
# проверка структуры файла (опечатки в значениях она не поймает)
docker compose config --quiet
# покажем итоговый файл с подставленными значениями и найдём ключевые строки
docker compose config | grep -E 'image:|single-node|127.0.0.1'
```

Читайте вывод глазами: опечатка вроде `single-nodeB` пройдёт проверку структуры, но сломает запуск.

---

## Шаг 8. Запуск: строго по порядку

Почему не всё сразу: Kibana и Logstash входят в Elasticsearch под своими пользователями, и пока мы их не создали, входить им нечем. Плюс Logstash должен писать в **уже готовый** data stream.

```bash
# загружаем значения из .env в оболочку, чтобы использовать $ELASTIC_PASSWORD в curl ниже
set -a; . ./.env; set +a
# запускаем только Elasticsearch
docker compose up -d elasticsearch
# ждём, пока он начнёт отвечать (пока порт закрыт, цикл повторяется)
until curl -s -o /dev/null -u "elastic:$ELASTIC_PASSWORD" http://127.0.0.1:9200; do echo ждём...; sleep 5; done
# должен вернуться JSON с номером версии 9.5.5
curl -s -u "elastic:$ELASTIC_PASSWORD" http://127.0.0.1:9200
```

Если цикл крутится дольше 3 минут: `docker compose logs --no-pager elasticsearch | tail -n 40` и `docker inspect --format '{{.RestartCount}}' elk-elasticsearch-1` (число перезапусков больше 0 значит контейнер падает).

Прогноз: что вернёт запрос без пароля (`curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9200`)? Проверьте.

### 8.1 Пользователи и роль

```bash
# задаём пароль служебному пользователю kibana_system (его использует Kibana)
curl -s -u "elastic:$ELASTIC_PASSWORD" -X POST http://127.0.0.1:9200/_security/user/kibana_system/_password -H 'Content-Type: application/json' -d "{\"password\":\"$KIBANA_PASSWORD\"}"
# создаём роль для Logstash: только запись в индексы filebeat-* и право на мониторинг кластера
curl -s -u "elastic:$ELASTIC_PASSWORD" -X PUT http://127.0.0.1:9200/_security/role/logstash_writer -H 'Content-Type: application/json' -d '{"cluster":["monitor"],"indices":[{"names":["filebeat-*"],"privileges":["write","create","create_index"]}]}'
# создаём пользователя logstash_internal с этой ролью
curl -s -u "elastic:$ELASTIC_PASSWORD" -X PUT http://127.0.0.1:9200/_security/user/logstash_internal -H 'Content-Type: application/json' -d "{\"password\":\"$LOGSTASH_INTERNAL_PASSWORD\",\"roles\":[\"logstash_writer\"]}"
```

Ожидаем ответы `{}` и `{"role":{"created":true}}` и `{"created":true}`.

### 8.2 Data stream для Filebeat (один раз)

```bash
# разово создаём шаблон и data stream filebeat-9.5.5; делаем под elastic (у служебных пользователей не хватает прав)
docker compose run --rm --no-deps filebeat setup --strict.perms=false --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["http://127.0.0.1:9200"]' -E output.elasticsearch.username=elastic -E output.elasticsearch.password="$ELASTIC_PASSWORD"
```

В конце ожидаем `Index setup finished.`

### 8.3 Остальное

```bash
# поднимаем Kibana, Logstash, Filebeat и node-exporter
docker compose up -d
# все пять сервисов должны быть Up
docker compose ps
# кто слушает порты (ожидаем 127.0.0.1:9200, 127.0.0.1:5601, 5044, 9100)
ss -tlnp | grep -E ':(9200|5601|5044|9100|9600)\b'
```

Kibana стартует 1-2 минуты. Проверка:

```bash
# ожидаем 200
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:5601/login
# смотрим логи, если не 200
docker compose logs --no-pager kibana | tail -n 20
```

---

## Шаг 9. Проверка потока логов внутри сервера

```bash
# создаём тестовую запись в системный журнал
logger "тест ELK с elk-vps"
# даём 10-20 секунд на путь Filebeat → Logstash → Elasticsearch и ищем запись
curl -s -u "elastic:$ELASTIC_PASSWORD" 'http://127.0.0.1:9200/filebeat-*/_search?q=message:ELK&size=1&pretty' | head -n 30
# документы в data stream считаются так
curl -s -u "elastic:$ELASTIC_PASSWORD" 'http://127.0.0.1:9200/_cat/indices/.ds-filebeat*?v'
```

Если пусто: смотрим по цепочке Filebeat → Logstash → Elasticsearch: `docker compose logs --no-pager filebeat | tail`, затем `logstash`.

Прогноз: что вернёт `curl http://195.208.61.11:5044` с **ноутбука** (порт 5044 в ufw открыт только для B)? Теперь проверьте.

---

## Шаг 10. nginx и HTTPS

Зачем nginx: Kibana слушает только 127.0.0.1:5601, это безопасно. Снаружи входит nginx на 80/443, расшифровывает HTTPS и передаёт запросы Kibana («администратор на проходной»). Авторизация: вход по логину и паролю Kibana (пользователи Elasticsearch).

```bash
# ставим nginx и certbot (программа получает сертификаты Let's Encrypt) с плагином для nginx
apt install -y nginx certbot python3-certbot-nginx
# убираем сайт-заглушку по умолчанию
rm -f /etc/nginx/sites-enabled/default
# создаём конфиг сайта
vim /etc/nginx/sites-available/kibana
```

Содержимое:

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name kibana.artemdevops.ru;

    location / {
        proxy_pass http://127.0.0.1:5601;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 120s;
    }
}
```

Что здесь что: `server_name` определяет, какой домен обслуживает блок; `proxy_pass` передаёт запрос Kibana; `Host` и `X-Forwarded-*` сообщают Kibana, кто и как пришёл (иначе в ссылках и логах будет 127.0.0.1); `Upgrade` и `Connection` нужны для WebSocket; `map` включает только эти два заголовка, когда клиент просит WebSocket.

```bash
# включаем сайт (ссылкой из sites-available в sites-enabled)
ln -s /etc/nginx/sites-available/kibana /etc/nginx/sites-enabled/kibana
# проверяем конфиг
nginx -t
# перечитываем конфиг без остановки сайта
systemctl reload nginx
# проверка по HTTP с самого сервера (ожидаем 302)
curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: kibana.artemdevops.ru' http://127.0.0.1/
# выпускаем сертификат, certbot сам допишет блок HTTPS и редирект (подставьте почту)
certbot --nginx -d kibana.artemdevops.ru --non-interactive --agree-tos -m <ваша-почта> --redirect
# проверяем автопродление сертификата (пробный запуск)
certbot renew --dry-run
# смотрим, что certbot дописал в конфиг
cat /etc/nginx/sites-available/kibana
```

Не повторяйте выпуск сертификата много раз подряд при ошибках: у Let's Encrypt лимит неудачных попыток. Сначала найдите причину (DNS, порт 80, имя сайта).

Откройте `https://kibana.artemdevops.ru`: форма входа Kibana. Логин `elastic`, пароль из `.env`.

В Kibana: меню → Stack Management → Data Views → Create → шаблон `filebeat-*`, поле времени `@timestamp`. Затем Discover: видны записи, в том числе тестовая.

Копия конфига nginx в репозиторий (чтобы не потерять):

```bash
# копируем рабочий конфиг в репозиторий
cp /etc/nginx/sites-available/kibana /opt/monitoring-compose/elk/nginx/kibana.conf
```

---

## Шаг 11. Пользователь только для просмотра (для друга)

Под `elastic` (суперпользователь) чужому входить нельзя.

```bash
# вводим пароль без показа на экране
read -rsp 'Пароль для друга: ' FRIEND_PWD; echo
# создаём пользователя с готовой ролью viewer (только чтение)
curl -s -u "elastic:$ELASTIC_PASSWORD" -X PUT http://127.0.0.1:9200/_security/user/andrey -H 'Content-Type: application/json' -d "{\"password\":\"$FRIEND_PWD\",\"roles\":[\"viewer\"],\"full_name\":\"Andrey (viewer)\"}"
# стираем переменную
unset FRIEND_PWD
```

Проверка: выйдите из Kibana, войдите как `andrey`; Discover открывается, а Stack Management (создание пользователей) недоступен. Если роль `viewer` не даёт нужного (нет данных в Discover или нет доступа к Kibana), напишите в чат: поправим роль.

---

## Шаг 12. Сквозная проверка с сервером B

Выполняется после того, как на B запущен стек (памятка B).

```bash
# какие серверы присылают логи; ожидаем два значения: elk-vps и monitoring-vps
curl -s -u "elastic:$ELASTIC_PASSWORD" 'http://127.0.0.1:9200/filebeat-*/_search?size=0' -H 'Content-Type: application/json' -d '{"aggs":{"hosts":{"terms":{"field":"host.name"}}}}'
# кто подключён к порту 5044 (должен быть IP сервера B)
ss -tn state established '( sport = :5044 )'
```

В Kibana (Discover, `filebeat-*`) в поле `host.name` должны быть `elk-vps` и `monitoring-vps`.

---

## Шаг 13. Перезагрузка

```bash
# перезагружаем сервер
reboot
```

После входа: `cd /opt/monitoring-compose/elk && docker compose ps`: все Up без вашего вмешательства; сайт открывается; свежая запись `logger "после ребута"` доходит до Kibana. Запишите, сколько времени всё поднималось и сколько памяти (`free -h`).

---

## Шаг 14. Фиксация в репозитории (после того как всё проверено)

```bash
cd /opt/monitoring-compose
# имя и почта для коммитов (только для этого репозитория)
git config user.name "Artyom"
git config user.email "<почта GitHub>"
# что изменилось: .env в списке быть не должно
git status
# добавляем всё нужное и проверяем ещё раз
git add .
git status
# коммит
git commit -m "ELK: compose, pipeline, filebeat, nginx"
```

Для отправки нужен доступ на запись. Самый простой путь на время обучения: deploy key.

```bash
# создаём ключ без пароля
ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -N "" -C "elk-vps"
# показываем открытую часть: вставьте её на GitHub: репозиторий → Settings → Deploy keys → Add, галочка Allow write access
cat ~/.ssh/github_deploy.pub
# SSH должен использовать этот ключ для github.com
printf 'Host github.com\n  IdentityFile ~/.ssh/github_deploy\n  IdentitiesOnly yes\n' >> ~/.ssh/config
# переключаем адрес репозитория на SSH
git remote set-url origin git@github.com:<ваш-логин>/monitoring-compose.git
# проверка доступа (в ответе: successfully authenticated)
ssh -T git@github.com
# забираем изменения сервера B, если он успел отправить свои, и кладём наши поверх
git pull --rebase origin main
# отправляем
git push origin main
```

---

## Что записывать для финальной памятки

Для каждой проблемы: шаг, команда, вывод, что помогло. Отдельно:
- через сколько минут заработал DNS и прошёл certbot;
- сколько занял запуск Kibana, общее время «с нуля до работающего стека»;
- сработала ли роль `viewer`;
- свободная память в покое (`free -h`);
- что не получилось с Tab/vim и что вы поправили.

## Типичные ошибки

| Симптом | Причина и действие |
|---|---|
| Elasticsearch сразу падает | `docker compose logs --no-pager elasticsearch`; чаще всего не применён `vm.max_map_count` (шаг 3) или опечатка в значении compose |
| Kibana: «server is not ready» | неверный пароль `kibana_system` или Elasticsearch ещё не готов; смотрите `docker compose logs --no-pager kibana` |
| Logstash: 401/403 | пользователь/роль не созданы (8.1) или другой пароль в `.env`; после правки `.env` нужен `docker compose up -d` |
| Данные попали в обычный индекс, а не в data stream | Logstash запущен до `filebeat setup` (8.2) |
| certbot: Timeout / connection refused | DNS ещё не доехал, закрыт порт 80, имя в `server_name` не совпадает |
| 502 Bad Gateway | Kibana ещё стартует или упала |
| Контейнер Filebeat: «config file must be owned…» | должен быть `--strict.perms=false` в `command`, проверьте отступы в compose |

## Известные ограничения

- Логи от B идут к A по интернету открытым текстом (без TLS на 5044), защита только файрвол по IP.
- Метрики A забирает Prometheus с B по интернету открытым текстом (порт 9100), тоже защита по IP.
- Внутри сети Docker TLS отключён (учебное упрощение).
- Нет политики хранения логов (ILM), нет защищённой очереди Logstash, нет алертов.
