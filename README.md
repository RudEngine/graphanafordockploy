# Grafana для HTTP-запросов в Dokploy

Traefik (Dokploy) → `access.log` (JSON) → **Alloy** → **Loki** → **Grafana** с готовым дашбордом.

```
docker-compose.yml
loki/loki.yaml                          # хранение 30 дней, single binary
alloy/config.alloy                      # читает access.log, ставит лейблы host/status
grafana/provisioning/...                # datasource Loki + провайдер дашбордов
grafana/dashboards/traefik-requests.json
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
5. **Deploy** → Grafana → Dashboards → Dokploy → **Dokploy · HTTP-запросы**. Логин `admin`.

Без git: тип Raw, конфиги добавить через Advanced → Mounts → File Mount, а в compose заменить `loki/loki.yaml` на `../files/loki.yaml` и т.д. (Dokploy кладёт file mounts в `../files/`).

## Что на дашборде

- Счётчики: запросы, 4xx, 5xx, уникальные IP, p95 времени ответа
- Запросы по классам статусов (2xx/3xx/4xx/5xx) и время ответа p50/p95/p99
- Топ путей (без query string), топ IP, ошибки 4xx/5xx по путям — клик по пути/IP фильтрует логи
- **Все запросы**: `IP  метод  хост+путь  →  статус  время  User-Agent`, раскрытие строки — все поля Traefik

Фильтры сверху: хост, статус, путь (regex, подстрока), IP (префикс).

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
- Срок хранения: `retention_period` в `loki/loki.yaml` (сейчас 30 дней).
