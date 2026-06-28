## Ansible Playbook: Установка ClickHouse и Vector

Этот playbook предназначен для автоматизированной установки и базовой настройки двух компонентов:

- **ClickHouse** — аналитическая СУБД
- **Vector** — агент сбора и доставки логов

Playbook рассчитан на выполнение в окружении prod.yml, где хосты запускаются в **Docker**‑контейнерах на базе **FedoraOS**.

---

#### Структура playbook

Playbook состоит из двух отдельных **plays**:

- **Install Clickhouse**
- **Install and configure Vector**

Каждый play выполняется на своей группе хостов, определённой в `inventory/prod.yml`.

#### Play 1: Install ClickHouse

**Что делает**
Скачивает **RPM**‑пакеты **ClickHouse** (client, server, common-static)

Использует **fallback‑URL** для пакета `clickhouse-common-static`, если основной URL недоступен

Устанавливает все скачанные **RPM**‑пакеты

Пропускает запуск сервиса ClickHouse (в Docker нет systemd)

Пропускает создание базы данных (сервер не запущен)

**Переменные**
Определены в `group_vars/clickhouse/vars.yml`:

```
clickhouse_version: "22.3.3.44"
clickhouse_packages:
    - clickhouse-client
    - clickhouse-server
    - clickhouse-common-static
```

**Особенности**

- Установка **RPM** выполняется через `with_fileglob`, чтобы корректно работать в Docker‑контейнере.
- Сервис ClickHouse не запускается, так как systemd отсутствует.
- Создание базы данных заменено на debug‑сообщение.Создание базы данных заменено на debug‑сообщение.

#### Play 2: Install and configure Vector

**Что делает**

- Создаёт необходимые директории (/etc/vector, /opt/vector)
- Скачивает архив Vector

- Распаковывает его в /opt/vector
- Разворачивает конфигурационный файл из шаблона

- Пропускает установку systemd‑unit и запуск сервиса (Docker)

**Переменные**

Определены в group_vars/vector/vars.yml:

```
vector_version: "0.30.0"
```

**Особенности**

- Все операции выполняются от root (sudo не используется)
- Сервис Vector не запускается, так как systemd отсутствует

- Handler перезапуска заменён на debug‑сообщение

#### Идемпотентность

Playbook полностью идемпотентен:

- При первом запуске скачивает и устанавливает необходимые компоненты
- При повторном запуске (--diff) не вносит изменений

- Все задачи переходят в состояние ok, changed=0

#### Параметры запуска

**Основной запуск**

```
ansible-playbook site.yml -i inventory/prod.yml
```

**Проверка изменений**

```
ansible-playbook site.yml -i inventory/prod.yml --diff
```

**Проверка идемпотентности**

```
ansible-playbook site.yml -i inventory/prod.yml --diff
```

#### Теги

В текущей версии playbook теги не используются, так как структура небольшая и plays разделены логически.
При необходимости теги могут быть добавлены, например:

`clickhouse`

`vector`

`install`

`config`
