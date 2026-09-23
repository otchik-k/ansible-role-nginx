# Роль nginx (самодостаточная)

Устанавливает nginx, настраивает reverse proxy на бэкенд контейнерного
приложения, прокси-кеш, gzip и раздачу статики/загрузок с диска.
Не требует ansible-galaxy и внешних ролей — все задачи в `tasks/main.yml`,
все шаблоны в `templates/`.

## Подключение

Скопируйте папку `nginx/` в `roles/` вашего проекта и добавьте роль в плейбук:

```yaml
- name: Deploy app
  hosts: my_hosts
  become: true
  roles:
    - configure-srv
    - deploy-sql
    - s3-storage
    - deploy-app
    - nginx
```

## Настройка

Конфигурация проекта — в файле `roles/nginx/vars/main.yml`. Это единственный
файл, который нужно править: у `vars/` наивысший приоритет в Ansible, поэтому
создавать `group_vars` и переопределять переменные снаружи не требуется.

```yaml
# roles/nginx/vars/main.yml
nginx_upstream_servers: ["127.0.0.1:8080"]   # адрес бэкенд-контейнера
nginx_server_name: "myapp.example.com"       # ваш домен или IP сервера
nginx_app_static_root: "/srv/app/public"     # volume со статикой фронта
nginx_app_media_root: "/srv/app/media"       # volume с загрузками
nginx_client_max_body_size: "128m"           # лимит размера загрузки
```

Технические значения по умолчанию (worker_processes, gzip, пути) лежат в
`defaults/main.yml` и обычно не требуют изменений. Если когда-нибудь понадобится
разовое переопределение «снаружи» — используйте extra-vars:
`ansible-playbook ... -e nginx_server_name=test.example.com`.

Ключевые переменные:

| Переменная | Назначение |
|---|---|
| `nginx_upstream_servers` | адреса бэкенд-контейнера (host:port) |
| `nginx_server_name` | домен/IP виртуального хоста |
| `nginx_app_static_root` / `nginx_app_media_root` | каталоги из docker volumes |
| `nginx_client_max_body_size` | лимит загрузок |
| `nginx_proxy_cache_*` | размер зоны/TTL прокси-кеша |
| `nginx_listen_ipv6` | слушать ли [::]:80 |

## Требования к хосту

- Debian/Ubuntu или RHEL/CentOS/Rocky/AlmaLinux (роль ставит пакет `nginx`).
- Статика и медиа контейнера расшарены на хост volume-ами:
  `./public:/srv/app/public`, `./media:/srv/app/media`.
