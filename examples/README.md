# Пример: Nginx как reverse proxy перед контейнерным приложением

Сценарий: приложение работает в контейнере (docker-compose), бэкенд внутри
контейнера отдаёт API и статику фронтенда. Nginx ставится на хост через роль
`geerlingguy.nginx` и решает три задачи:

1. **Reverse proxy** — весь трафик на 80/443 идёт через nginx, который
   проксирует запросы на бэкенд (`proxy_pass http://app_backend`).
2. **Кеширование** — прокси-кеш (`proxy_cache`) для ответов бэкенда +
   браузерное кеширование (`expires` / `Cache-Control`) для статики и загрузок.
3. **Раздача статики/загрузок напрямую с диска** — файлы фронтенда и
   пользовательские загрузки монтируются из контейнера volume-ами на хост, и
   nginx отдаёт их сам, не дёргая бэкенд.

## Файлы

| Файл                     | Назначение                                   |
|--------------------------|----------------------------------------------|
| `requirements.yml`       | Зависимость роли из Galaxy                   |
| `inventory`              | Хост(а) с nginx                              |
| `group_vars/web.yml`     | Все настройки: upstream, кеш, vhost, статика |
| `playbook.yml`           | Запуск роли + создание каталога кеша          |

## Запуск

```bash
ansible-galaxy install -r examples/requirements.yml
ansible-playbook -i examples/inventory examples/playbook.yml
```

Перед запуском подставьте:
- `server_name` — ваш домен;
- адрес upstream (`backend:8080`) — имя сервиса и порт из docker-compose;
- пути `root` (`/srv/app/public`) и `alias` (`/srv/app/media/`) — точки
  монтирования volume'ов со статикой и загрузками.

## Пример docker-compose (фрагмент)

```yaml
services:
  backend:
    build: ./backend
    expose:
      - "8080"
    volumes:
      - static_data:/app/public:ro     # сборка фронтенда
      - media_data:/app/media          # загрузки пользователей

volumes:
  static_data:
  media_data:
```

Чтобы nginx «видел» эти файлы на хосте, примонтируйте том в ту же точку на
хосте (например, `/srv/app/public` и `/srv/app/media` — тогда достаточно
изменить volume на bind-mount: `./data/public:/app/public`).

Если nginx запущен **внутри той же docker-сети**, укажите в upstream имя
сервиса (`backend:8080`). Если nginx на хосте — проброшенный порт
(`127.0.0.1:8080`) или IP контейнера.

## Альтернатива: nginx в отдельном контейнере

Если nginx тоже должен быть контейнером, используйте образ `nginx:alpine` и
смонтируйте сгенерированные ролью конфиги либо свой конфигурационный файл:

```yaml
  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/app.conf:/etc/nginx/conf.d/default.conf:ro
      - ./data/public:/srv/app/public:ro
      - ./data/media:/srv/app/media:ro
      - nginx_cache:/var/cache/nginx
    networks: [app]
```

Содержимое `app.conf` можно взять из сгенерированного
`/etc/nginx/conf.d/app.conf` на хосте после прогона плейбука (блок `server`
переносится один в один; `upstream` и `proxy_cache_path` кладутся в
`http`-контекст через `/etc/nginx/conf.d/` или `nginx.conf`).

## Как проверить кеширование

```bash
curl -sI https://app.example.com/api/status | grep X-Cache-Status
# MISS -> HIT при повторном запросе (для некешируемых с куками будет BYPASS)
```
