# Описание ролей

В проекте используются две Ansible-роли: **clickhouse** (взятая из Ansible Galaxy, автор Alexey V. Bobrov), **vector-role** (кастомная), **lighthouse** (кастомная). Все предназначены для установки и конфигурирования соответствующих сервисов на целевых хостах.

---

## 1. clickhouse

Роль для установки, настройки и управления ClickHouse (server + client) на системах семейства Debian и RedHat.

### Поддерживаемые ОС

| Дистрибутив      | Версии                            |
| ---------------- | --------------------------------- |
| Ubuntu           | xenial, bionic, focal             |
| Debian           | jessie, stretch, buster, bullseye |
| CentOS/RHEL (EL) | 7, 8                              |

### Как работает роль

1. Определяет переменные в зависимости от `ansible_os_family` (`debian.yml` или `redhat.yml`)
2. Проверки: проверяет что версия ClickHouse доступна
3. Установка: через доступный `ansible_pkg_mgr` (`apt` / `yum` / `dnf`)
4. Конфигурация: генерирует конфигурационные файлы по шаблонам (`config.j2`, `users.j2`, `zookeeper-servers.j2`, `remote_servers.j2`, `dicts.j2`)
5. Сервис: запускает/перезапускает службу `clickhouse-server`
6. Ждёт пока сервер станет доступен (проверка порта)
7. Настройка БД, словарей
8. Удаление: при включённом флаге `clickhouse_remove`

### Переменные (defaults)

| Переменная                                 | Значение по умолчанию             | Описание                                                                          |
| ------------------------------------------ | --------------------------------- | --------------------------------------------------------------------------------- |
| `clickhouse_version`                       | `'latest'`                        | Версия ClickHouse для установки                                                   |
| `clickhouse_service_ensure`                | `'started'`                       | Состояние сервиса: `started` / `stopped`                                          |
| `clickhouse_service_enable`                | `true`                            | Автозапуск при загрузке системы                                                   |
| `clickhouse_setup`                         | `'package'`                       | Способ установки: `package` / `source`                                            |
| `clickhouse_remove`                        | `false`                           | Флаг удаления сервиса                                                             |
| `clickhouse_remove_full`                   | `false`                           | Полное удаление (DB + config), требует remove=true                                |
| `clickhouse_ready_retries`                 | `3`                               | Количество попыток проверки готовности                                            |
| `clickhouse_ready_delay`                   | `5`                               | Задержка между проверками (сек.)                                                  |
| `clickhouse_profiles_default`              | _(см. defaults)_                  | Профили ClickHouse по умолчанию                                                   |
| `clickhouse_profiles_custom`               | `{}`                              | Пользовательские профили                                                          |
| `clickhouse_http_port`                     | `8123`                            | HTTP порт                                                                         |
| `clickhouse_tcp_port`                      | `9000`                            | TCP порт                                                                          |
| `clickhouse_interserver_http`              | `9009`                            | Межсерверный HTTP порт                                                            |
| `clickhouse_ssl_server`                    | _(см. defaults)_                  | Параметры SSL-сертификатов сервера                                                |
| `clickhouse_ssl_client`                    | _(см. defaults)_                  | Параметры SSL-клиента                                                             |
| `clickhouse_listen_host`                   | `["::1", "127.0.0.1"]`            | Список адресов для прослушивания                                                  |
| `clickhouse_users_default`                 | `default`, `readonly`             | Пользователи по умолчанию                                                         |
| `clickhouse_users_custom`                  | `{}`                              | Пользовательские пользователи                                                     |
| `clickhouse_quotas_default`                | `default`                         | Квоты по умолчанию                                                                |
| `clickhouse_quotas_custom`                 | `{}`                              | Пользовательские квоты                                                            |
| `clickhouse_dbs_default`                   | `[]`                              | Базы данных по умолчанию                                                          |
| `clickhouse_dbs_custom`                    | `[]`                              | Пользовательские базы данных                                                      |
| `clickhouse_config`                        | _(см. defaults)_                  | Основные параметры конфигурации (max_connections, uncompressed_cache_size и т.д.) |
| `clickhouse_dicts`                         | `[]`                              | Внешние словари                                                                   |
| `clickhouse_kafka_config`                  | `[]`                              | Настройки Kafka-коннектора                                                        |
| `clickhouse_kafka_topics_config`           | `[]`                              | Конфигурация топиков Kafka                                                        |
| `clickhouse_merge_tree_config`             | `[]`                              | Параметры MergeTree                                                               |
| `clickhouse_mlock_status`                  | `false`                           | Включение mlock                                                                   |
| `clickhouse_logger`                        | _(см. defaults)_                  | Настройки логирования                                                             |
| `clickhouse_config_dictionaries_lazy_load` | `true`                            | Ленивая загрузка встроенных словарей                                              |
| `clickhouse_path_base`                     | `/var/lib`                        | Базовый путь                                                                      |
| `clickhouse_path_configdir`                | `/etc/clickhouse-server`          | Путь к конфигурационным файлам                                                    |
| `clickhouse_path_logdir`                   | `/var/log/clickhouse-server`      | Путь к логу                                                                       |
| `clickhouse_path_data`                     | `/var/lib/clickhouse/`            | Путь к данным                                                                     |
| `clickhouse_path_user_files`               | `/var/lib/clickhouse/user_files/` | Путь к пользовательским файлам                                                    |
| `clickhouse_path_tmp`                      | `/var/lib/clickhouse/tmp/`        | Путь к временным файлам                                                           |

### Структура tasks

```
tasks/main.yml
├── precheck.yml          # Проверки перед установкой
├── params.yml            # Применение параметров OS-specific
├── install/<pkg_mgr>.yml # Установка пакета
├── configure/sys.yml     # Развёртка основного конфига
├── service.yml           # Управление сервисом
├── configure/db.yml      # Создание баз данных
├── configure/dict.yml    # Настройка словарей
└── remove.yml            # Удаление (если clickedhouse_remove=true)
```

### Теги

- `always` — переменные, проверки, параметры
- `install` — установка
- `config` / `config_sys` — системная конфигурация
- `config_db` — создание БД
- `config_dict` — словари
- `remove` — удаление

### Handler

- `Restart Clickhouse Service` — через `set_fact` переключает `clickhouse_service_ensure` в `restarted`, слушает событие `restart-ch`

---

## 2. vector-role

Кастомная роль для автоматической установки и конфигурирования Vector — высокопроизводительного коллектора, трансформера и транспортера логов.

### Поддерживаемые ОС

RedHat-совместимые дистрибутивы с `yum` (CentOS, RHEL, AlmaLinux, Rocky и др.)

### Как работает роль

1. Создаёт группу и пользователя `vector`
2. Добавляет пользователя `vector` в группу `adm` (для чтения логов)
3. Создаёт необходимые директории (`/opt/vector-*`, `/etc/vector`, `/var/lib/vector`, `/var/log/vector`)
4. Скачивает бинарный архив с официальных пакетов Timber.io
5. Распаковывает архив, копирует бинарник в `/usr/local/bin/vector`
6. Восстанавливает SELinux контексты
7. Разворачивает конфигурационный файл (`vector.toml`) и systemd unit (`vector.service`)
8. Перезагружает systemd, включает и запускает сервис
9. Проверяет версию установленного Vector

### Переменные (defaults)

| Переменная       | Значение   | Описание      |
| ---------------- | ---------- | ------------- |
| `vector_version` | `"0.55.0"` | Версия Vector |

### Переменные (vars)

| Переменная           | Значение                           | Описание                       |
| -------------------- | ---------------------------------- | ------------------------------ |
| `vector_repo_url`    | URL бинарного tar.gz               | Ссылка для скачивания          |
| `vector_config_dir`  | `/etc/vector`                      | Путь к конфигам                |
| `vector_config_path` | `/etc/vector/vector.toml`          | Полный путь к главному конфигу |
| `vector_install_dir` | `/opt/vector-{{ vector_version }}` | Папка распаковки архива        |
| `vector_data_dir`    | `/var/lib/vector`                  | Папка данных                   |
| `vector_log_dir`     | `/var/log/vector`                  | Папка логов                    |
| `vector_user`        | `vector`                           | Имя пользователя               |
| `vector_group`       | `vector`                           | Имя группы                     |

### Шаблоны

- **`templates/vector.toml.j2`** — основной конфигурационный файл Vector (источники, пайплайны, приёмники)
- **`templates/vector.service.j2`** — systemd unit-файл для Vector

### Структура tasks

```
tasks/main.yml
├── create vector group          # ansible.builtin.group
├── create vector user           # ansible.builtin.user
├── add vector to adm group      # ansible.builtin.user
├── create directories           # ansible.builtin.file (loop)
├── download archive             # ansible.builtin.get_url
├── unpack vector                # ansible.builtin.unarchive
├── install binary               # ansible.builtin.copy
├── restore selinux contexts     # ansible.builtin.command
├── deploy config                # ansible.builtin.template → notify: restart vector service
├── deploy vector config         # ansible.builtin.template → notify: restart vector service
├── reload systemd               # ansible.builtin.systemd (daemon_reload)
├── enable and start vector      # ansible.builtin.systemd
├── verify vector installation   # block: check version + debug
└── check vector service status  # ansible.builtin.systemd
```

### Handlers

| Handler                  | Действие                                                  |
| ------------------------ | --------------------------------------------------------- |
| `restart vector service` | Перезапуск сервиса Vector через `ansible.builtin.service` |

Handler вызывается автоматически при изменениях в задачах `deploy config` (конфиг Vector) и `deploy vector config` (systemd unit) — используется механизм `notify`.

### Использование в плейбуке

Плейбук `playbook/site.yml` включает роль через `include_role`:

```yaml
- name: Intalling vector
  hosts: vector
  tasks:
    - name: Install dependencies
      ansible.builtin.yum:
        name: ["wget", "tar", "gzip"]
        state: present
      become: true

    - name: Include role
      ansible.builtin.include_role:
        name: "{{ 'vector-role' }}"
```

Зависимости (wget, tar, gzip) устанавливаются до включения роли, поскольку они требуются для загрузки архива Vector.

---

## 3. lighthouse

Кастомная роль для установки и запуска **Lighthouse CI Server** — инструмента от Google для мониторинга производительности веб-страниц и отслеживания изменений метрик Web Vitals. Роль устанавливает Node.js, Lighthouse CLI Server и разворачивает веб-интерфейс с поддержкой reverse-proxy через nginx.

### Поддерживаемые ОС

| Дистрибутив                          | Версии |
| ------------------------------------ | ------ |
| CentOS/RHEL / AlmaLinux / Rocky (EL) | 8, 9   |

### Как работает роль

1. Создаёт системного пользователя и группу `lighthouse`
2. Устанавливает репозиторий Node.js через Nodesource (версия задаётся переменной)
3. Устанавливает Node.js и npm
4. Устанавливает `@lhci/cli` глобально через npm
5. Создаёт необходимые директории (проект, загрузка артефактов, хранение артефактов)
6. Разворачивает systemd unit-файл для сервиса Lighthouse
7. Запускает и включает сервис Lighthouse
8. Опционально устанавливает и настраивает Nginx как reverse-proxy с SSL

### Переменные (defaults)

| Переменная                             | Значение по умолчанию                            | Описание                                  |
| -------------------------------------- | ------------------------------------------------ | ----------------------------------------- |
| `lighthouse_nodejs_version`            | `"18"`                                           | Версия Node.js для установки (18, 20)     |
| `lighthouse_app_version`               | `"latest"`                                       | Версия @lhci/cli для установки            |
| `lighthouse_service_ensure`            | `"started"`                                      | Состояние сервиса: `started` / `stopped`  |
| `lighthouse_service_enable`            | `true`                                           | Автозапуск при загрузке системы           |
| `lighthouse_database_type`             | `"sqlite"`                                       | Тип БД: `sqlite` или `postgresql`         |
| `lighthouse_db_name`                   | `"lighthouse"`                                   | Имя базы данных (для PostgreSQL)          |
| `lighthouse_db_user`                   | `"lighthouse"`                                   | Пользователь БД (для PostgreSQL)          |
| `lighthouse_db_password`               | `""`                                             | Пароль БД (для PostgreSQL)                |
| `lighthouse_db_host`                   | `"127.0.0.1"`                                    | Хост БД (для PostgreSQL)                  |
| `lighthouse_db_port`                   | `5432`                                           | Порт БД (для PostgreSQL)                  |
| `lighthouse_listen_host`               | `"0.0.0.0"`                                      | Адрес прослушивания сервера               |
| `lighthouse_listen_port`               | `9888`                                           | Порт веб-сервера Lighthouse               |
| `lighthouse_auth_token`                | `""`                                             | Токен аутентификации LHCI_TOKEN           |
| `lighthouse_basicauth_enabled`         | `false`                                          | Включение Basic Auth поверх LHCI Token    |
| `lighthouse_basicauth_username`        | `""`                                             | Имя пользователя для Basic Auth           |
| `lighthouse_basicauth_password`        | `""`                                             | Пароль для Basic Auth                     |
| `lighthouse_upload_dir`                | `/var/lib/lighthouse/storage/upload-dir`         | Папка для загруженных файлов              |
| `lighthouse_artifact_store_dir`        | `/var/lib/lighthouse/storage/artifact-store-dir` | Папка хранения артефактов                 |
| `lighthouse_project_dir`               | `/opt/lighthouse`                                | Основная директория проекта               |
| `lighthouse_install_nginx`             | `false`                                          | Установить Nginx в качестве reverse-proxy |
| `lighthouse_nginx_server_name`         | `"lighthouse.example.com"`                       | Имя хоста для Nginx                       |
| `lighthouse_nginx_ssl_enabled`         | `false`                                          | Включить SSL для Nginx                    |
| `lighthouse_nginx_ssl_certificate`     | `""`                                             | Путь к SSL-сертификату                    |
| `lighthouse_nginx_ssl_certificate_key` | `""`                                             | Путь к ключу SSL                          |

### Структура tasks

```
tasks/main.yml
├── user / group              # Создание пользователя 'lighthouse'
├── install nodejs            # Установка Node.js через Nodesource
├── install @lhci/cli         # Глобальная установка через npm
├── create directories        # Создание рабочих директорий
├── database setup            # Подготовка SQLite или PostgreSQL клиента
├── deploy systemd unit       # Шаблон lighthouse.service.j2 → notify: restart lighthouse
├── reload systemd            # daemon_reload
├── enable and start service  # Активация сервиса
└── nginx reverse proxy       # Опциональный блок: install nginx + config (when: lighthouse_install_nginx)
    ├── install nginx
    ├── deploy nginx config    # Шаблон lighthouse-nginx.conf.j2 → notify: reload nginx
    └── start nginx
```

### Шаблоны

- **`templates/lighthouse.service.j2`** — systemd unit-файл. Конфигурируется полностью через environment variables:
  - `LHCI_STORAGE_TYPE` / `LHCI_SQLITE_PATH` — SQLite режим
  - `LHCI_POSTGRES_CONNECTION_STRING` — PostgreSQL режим
  - `PORT`, `UPLOAD_DIR`, `ARTIFACT_STORE_DIR` — директории и порт
  - `LHCI_TOKEN` — токен аутентификации
  - `BASIC_AUTH_USERNAME` / `BASIC_AUTH_PASSWORD` — базовая аутентификация (опционально)
  - Использует `ExecStart=/usr/bin/lhci server`

- **`templates/lighthouse-nginx.conf.j2`** — конфигурация Nginx reverse-proxy:
  - Проксирует запросы на `127.0.0.1:{{ lighthouse_listen_port }}`
  - Поддерживает WebSocket upgrade (для Real Time Dashboard)
  - Настройный gzip compression
  - Healthcheck endpoint `/healthcheck`
  - Опциональный SSL (`listen 443 ssl http2`)
  - `client_max_body_size 100M` для загрузки крупных артефактов

### Handlers

| Handler              | Действие                                      |
| -------------------- | --------------------------------------------- |
| `restart lighthouse` | Перезапуск сервиса Lighthouse через `systemd` |
| `reload nginx`       | Грациозная перезагрузка Nginx (reload)        |
| `restart nginx`      | Полная перезагрузка Nginx (restart)           |

Handler `restart lighthouse` вызывается автоматически при изменении systemd unit-файла (через `notify`).

### Теги

- `always` — переменные
- `install` — установка зависимостей (Node.js, npm пакеты)
- `user` — создание пользователя/группы
- `config` — развёртка сервиса и настроек
- `service` — управление состоянием сервиса
- `database` — подготовка СУБД
- `nginx` — опциональная настройка Nginx

### Пример использования в плейбуке

```yaml
# ===== Третий play — Install Lighthouse CI Server =====

- name: Installing Lighthouse CI Server
  hosts: lighthouse
  become: true

  pre_tasks:
    - name: Update yum cache
      ansible.builtin.yum:
        update_cache: yes

  tasks:
    - name: Include role
      ansible.builtin.include_role:
        name: "{{ 'lighthouse' }}"
      vars:
        lighthouse_database_type: sqlite
        lighthouse_listen_port: 9888
        lighthouse_auth_token: "my-secret-token"

    - name: Enable Nginx reverse proxy with SSL
      ansible.builtin.set_fact:
        lighthouse_install_nginx: true
        lighthouse_nginx_server_name: "lighthouse.example.com"
        lighthouse_nginx_ssl_enabled: true
        lighthouse_nginx_ssl_certificate: "/etc/ssl/certs/lighthouse.crt"
        lighthouse_nginx_ssl_certificate_key: "/etc/ssl/private/lighthouse.key"
```

### Требования

- Целевой хост должен иметь доступ в интернет для загрузки репозитория Nodesource и пакетов
- Для PostgreSQL режима необходима внешняя или локальная база данных PostgreSQL
- Для HTTPS mode нужны действительные SSL-сертификаты
