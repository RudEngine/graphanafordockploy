# Grafana для HTTP-запросов и ресурсов контейнеров в Dokploy

- Traefik (Dokploy) → `access.log` (JSON) → **Alloy** → **Loki** → **Grafana**: дашборд HTTP-запросов.
- Docker → cAdvisor (встроен в **Alloy**) → **Prometheus** → **Grafana**: дашборд CPU/RAM по контейнерам.

```
docker-compose.yml
loki/loki.yaml                          # хранение 30 дней, single binary
alloy/config.alloy                      # access.log → Loki; cAdvisor + хост → Prometheus
grafana/provisioning/...                # datasources Loki и Prometheus + провайдер дашбордов
grafana/dashboards/traefik-requests.json
grafana/dashboards/docker-containers.json
grafana/home.json                       # домашняя страница: список наших дашбордов
```

## 1. Включить access-лог Traefik

На сервере в `/etc/dokploy/traefik/traefik.yml` должен быть блок ниже. Первые три ключа — ровно то, что пишет кнопка **Requests → Activate** в Dokploy; `fields` добавляет User-Agent и Referer (по умолчанию Traefik заголовки не пишет).

```yaml
accessLog:
  filePath: /etc/dokploy/traefik/dynamic/access.log
  format: json
  bufferingSize: 100
  fields:
    headers:
      defaultMode: drop
      names:
        User-Agent: keep
        Referer: keep
        # Cf-Connecting-Ip: keep   # если домен за Cloudflare — реальный IP клиента
```

Перезапусти Traefik (в Dokploy: Web Server → Traefik → Reload, или `docker restart dokploy-traefik`) и проверь:

```bash
tail -f /etc/dokploy/traefik/dynamic/access.log
```

> ⚠️ Если потом выключить и включить Requests кнопкой в Dokploy, он перезапишет блок без `fields` — User-Agent пропадёт, остальное продолжит работать.

## 2. Задеплоить стек

1. Залей эту папку в git (Gitea/GitHub).
2. Dokploy → проект → **Create Service → Compose** (тип Docker Compose), провайдер — твой git, Compose Path: `docker-compose.yml`.
3. **Environment**:
   ```
   GRAFANA_HOST=grafana.victory-pk.online
   GRAFANA_ADMIN_PASSWORD=<длинный пароль>
   ```
4. **Domains**: service `grafana`, port `3000`, host = `GRAFANA_HOST`, HTTPS + Let's Encrypt. A-запись на сервер.
5. **Deploy** → Grafana → Dashboards → Dokploy → **Dokploy · HTTP-запросы** и **Dokploy · Контейнеры**. Логин `admin`.

Без git: тип Raw, конфиги добавить через Advanced → Mounts → File Mount, а в compose заменить `loki/loki.yaml` на `../files/loki.yaml` и т.д. (Dokploy кладёт file mounts в `../files/`).

## Что на дашборде

- Счётчики: запросы, 4xx, 5xx, уникальные IP, p95 времени ответа
- Запросы по классам статусов (2xx/3xx/4xx/5xx) и время ответа p50/p95/p99
- Топ путей (без query string), топ IP, ошибки 4xx/5xx по путям — клик по пути/IP фильтрует логи
- **Все запросы**: `IP  метод  хост+путь  →  статус  время  User-Agent`, раскрытие строки — все поля Traefik

Фильтры сверху: хост, статус, путь (regex, подстрока), IP (префикс).

### Dokploy · Контейнеры

- Счётчики: CPU и RAM всего хоста, ядер / RAM, число контейнеров
- **CPU по контейнерам** и **RAM по контейнерам** — стек, легенда-таблица справа (среднее, максимум, текущее), отсортирована по среднему: сверху тот, кто больше всех ест
- **Сейчас** — таблица: контейнер, CPU, RAM, лимит RAM (если задан)
- Сеть и диск по контейнерам

CPU считается как в `docker stats`: 100% = одно ядро. Фильтр сверху — контейнер.
Имена: у compose-сервисов — имя контейнера (`project-service-1`), у приложений Dokploy (Swarm) хвост задачи `.1.<id>` отрезается, поэтому реплики и редеплои — одна линия.

## Полезные запросы в Explore

```logql
# Все POST в API
{job="traefik", host="api.victory-pk.online"} | json | RequestMethod="POST"

# Кто ходит в админку
{job="traefik", host="victory-pk.online"} | json | RequestPath=~"/admin.*"
  | line_format `{{.ClientHost}} {{.RequestMethod}} {{.RequestPath}} {{.DownstreamStatus}}`

# Медленнее 1 секунды (Duration в наносекундах)
{job="traefik"} | json | Duration > 1000000000

# Сканеры: запросы, не попавшие ни в один роутер (мусорный Host, голый IP)
{job="traefik", host="unrouted"}

# Топ IP по 4xx за час
topk(10, sum by (ClientHost) (count_over_time({job="traefik", status=~"4.."} | json ClientHost | keep ClientHost [1h])))
```

## Как устроено и почему так

- **Лейблы только `job`, `host`, `status`.** Путь, IP, метод, UA — внутри JSON и достаются через `| json`. Иначе каждый IP/путь стал бы отдельным стримом и Loki бы распух.
- **`host="unrouted"`** — все запросы, не совпавшие ни с одним роутером. Сканеры шлют в Host что угодно, без этого лейблы разрастутся. Оригинальный Host остаётся в строке лога.
- **Запросы самой Grafana отбрасываются** (по `GRAFANA_HOST`), иначе автообновление дашборда засоряет логи.
- **Ротация Dokploy не ломает сбор.** Dokploy по крону оставляет в `access.log` последние 1000 строк через `mv`. Поэтому монтируется папка, а не файл, а время берётся из `StartUTC` — при перечитывании хвоста Loki отбрасывает дубли.
- **Наружу торчит только Grafana** (через Traefik с HTTPS). У Loki и Alloy портов нет.
- Alloy монтирует `/etc/dokploy/traefik/dynamic` read-only. Там же лежит `acme.json` с TLS-ключами — контейнер его видеть может, но наружу не открыт. Если это критично, можно перенести `filePath` лога в отдельную папку, но тогда встроенная страница Requests в Dokploy перестанет его видеть.
- Срок хранения: `retention_period` в `loki/loki.yaml` (сейчас 30 дней); метрики — `--storage.tsdb.retention.time` у Prometheus в `docker-compose.yml` (30 дней).
- **Меню почищено** переменными `GF_*` в `docker-compose.yml`: выключены Drilldown-плагины, Advisor, Alerting, снапшоты, публичные дашборды, Help и лента новостей. Home открывает список наших дашбордов (по тегу `dokploy`). Bookmarks, Starred и пустой пункт Drilldown в Grafana 13 настройками не убираются, а Connections и Administration всегда видны админу.
- **Метрики контейнеров.** Prometheus сам ничего не скрейпит — Alloy пушит в него через remote write. В Prometheus уходит только то, что есть на дашборде (фильтр в `prometheus.relabel`), остальное cAdvisor плодит сотнями рядов.
- ⚠️ **Alloy получает `docker.sock`, `/run/containerd`, `/sys`, `/var/lib/docker`, `cgroup: host` и `pid: host`.** Сокет containerd нужен на Docker 29+ с хранилищем образов containerd (`docker info` → `storage=overlayfs`, по умолчанию при свежей установке): без него в логах Alloy `Registration of the docker container factory failed`, а на дашборде есть только хост. Без этого cAdvisor не видит чужие контейнеры и их сеть. Доступ к `docker.sock` (даже `:ro`) — это фактически root на хосте, так что Alloy наружу открывать нельзя (портов у него и нет).
