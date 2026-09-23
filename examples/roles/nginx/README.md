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

Все параметры — обычные ansible-переменные со значениями по умолчанию в
`defaults/main.yml`. Переопределите нужные в `group_vars/my_hosts.yml`:

```yaml
nginx_vhosts:
  - listen: "80"
    server_name: "myapp.example.com"
    filename: "myapp.conf"
    root: "/srv/app/public"
    extra_parameters: |
      ...                # см. defaults/main.yml как образец

nginx_upstreams:
  - name: app_backend
    servers: ["127.0.0.1:8080"]   # порт backend-контейнера
```

Ключевые переменные:

| Переменная | Назначение |
|---|---|
| `nginx_upstreams` | upstream на бэкенд-контейнер (keepalive) |
| `nginx_vhosts` | vhost'ы: статика, SPA-fallback, @backend с proxy_cache |
| `nginx_proxy_cache_path` | зона прокси-кеша (пустая строка = выключить) |
| `nginx_app_static_root` / `nginx_app_media_root` | каталоги из docker volumes |
| `nginx_extra_http_options` | gzip и опции http {} |
| `nginx_client_max_body_size` | лимит загрузок |

## Требования к хосту

- Debian/Ubuntu или RHEL/CentOS/Rocky/AlmaLinux (роль ставит пакет `nginx`).
- Статика и медиа контейнера расшарены на хост volume-ами:
  `./public:/srv/app/public`, `./media:/srv/app/media`.
