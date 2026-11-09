# ClickHouse Observability Stack

Локальный стек для экспериментов с ClickHouse-кластером, сбора метрик и событий, а также проверки интеграции ClickHouse MCP с Open WebUI и Ollama.

Стек запускался на Windows с помощью Docker Desktop. Ollama установлена на Windows-хосте отдельно от Docker Compose.

## Состав стека

- **ClickHouse cluster** — четыре узла, настроенные как 2 шарда × 2 реплики.
- **ClickHouse Observability** — отдельный экземпляр ClickHouse для данных наблюдаемости.
- **Apache NiFi** — потоки для сбора данных и имитации нагрузки на кластер.
- **ClickHouse MCP** — MCP-сервис для подключения AI-инструментов к ClickHouse.
- **Open WebUI** — веб-интерфейс для работы с языковыми моделями и инструментами.
- **Ollama** — запущенная на Windows локальная платформа для языковых моделей.

## Требования

- Windows и Docker Desktop с запущенным Docker Engine.
- Ollama, установленная и запущенная на Windows, если планируется использовать локальные модели.
- Свободные порты, перечисленные в разделе [Адреса и порты](#адреса-и-порты).

## Запуск

Откройте PowerShell в каталоге проекта и запустите сервисы:

```powershell
docker compose up -d --build
```

Проверьте состояние контейнеров:

```powershell
docker compose ps
```

## Обязательная настройка NiFi после запуска

После запуска контейнера NiFi обязательно укажите путь к файлу потока в `nifi.properties`. Выполните команду в PowerShell:

```powershell
docker exec -u nifi -it nifi bash -c 'source /opt/nifi/scripts/common.sh && prop_replace "nifi.flow.configuration.file" "/opt/nifi/nifi-current/flow/flow.json.gz"'
```

Затем откройте NiFi в браузере по адресу [https://localhost:8443](https://localhost:8443) и войдите с учётными данными из `.env`.

В интерфейсе NiFi щёлкните правой кнопкой мыши по холсту и запустите все потоки (**Start**). Потоки:

- собирают метрики и события из ClickHouse-кластера и передают их в отдельный ClickHouse Observability;
- имитируют нагрузку на кластер, выполняя запросы `INSERT` и `SELECT`.

## Адреса и порты


| Сервис             | Адрес с Windows-хоста | Назначение                                                  |
| ------------------------ | -------------------------------- | --------------------------------------------------------------------- |
| ClickHouse 01            | `http://localhost:18123`         | HTTP-интерфейс узла 01                                   |
| ClickHouse 02            | `http://localhost:18124`         | HTTP-интерфейс узла 02                                   |
| ClickHouse 03            | `http://localhost:18125`         | HTTP-интерфейс узла 03                                   |
| ClickHouse 04            | `http://localhost:18126`         | HTTP-интерфейс узла 04                                   |
| ClickHouse Observability | `http://localhost:18127`         | HTTP-интерфейс хранилища наблюдаемости |
| ClickHouse MCP           | `http://localhost:8000`          | MCP-сервис                                                      |
| Open WebUI               | `http://localhost:3000`          | Веб-интерфейс                                             |
| NiFi                     | `https://localhost:8443`         | Управление потоками                                 |

Нативные TCP-порты ClickHouse, опубликованные на хосте: `19000`–`19004` для узлов 01–04 и Observability соответственно.

Внутри сети Docker ClickHouse MCP подключается к ClickHouse Observability по адресу `clickhouse-observability:8123`.

## Учётные данные

Значения по умолчанию из `.env`:


| Сервис             | Пользователь | Пароль      |
| ------------------------ | ------------------------ | ----------------- |
| ClickHouse cluster       | `admin`                  | `admin`           |
| ClickHouse Observability | `admin`                  | `admin`           |
| NiFi                     | `admin`                  | `adminadminadmin` |

Пароль NiFi должен содержать **более 12 символов**. Текущее значение `adminadminadmin` соответствует этому требованию.

При необходимости измените логины и пароли непосредственно в `.env` до запуска контейнеров. Не публикуйте файл `.env` и не используйте приведённые значения по умолчанию в общедоступном или production-окружении.

Порт MCP также настраивается в `.env` через `MCP_PORT` (по умолчанию `8000`).

## ClickHouse MCP и Open WebUI

В проект включён ClickHouse MCP, который собирается из каталога `clickhouse-mcp` и подключается к ClickHouse Observability. Инструкцию по подключению ClickHouse MCP к Open WebUI смотрите в [официальной документации ClickHouse](https://clickhouse.com/docs/ru/guides/use-cases/ai-ml/MCP/open-webui).

Ollama установлена на Windows-хосте отдельно. Настройте её как модельный backend в Open WebUI согласно используемому вами способу запуска и сетевой конфигурации.

### Подключение ClickHouse MCP к Claude Desktop

ClickHouse MCP можно подключить и к Claude Desktop. Для этого установите `uv`, затем добавьте конфигурацию сервера в файл `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "mcp-clickhouse": {
      "command": "uv",
      "args": [
        "run",
        "--with",
        "mcp-clickhouse",
        "--python",
        "3.10",
        "mcp-clickhouse"
      ],
      "env": {
        "CLICKHOUSE_HOST": "localhost",
        "CLICKHOUSE_USER": "admin",
        "CLICKHOUSE_PORT": "18127",
        "CLICKHOUSE_PASSWORD": "admin",
        "CLICKHOUSE_SECURE": "false"
      }
    }
  }
}
```

Этот конфиг подключает Claude Desktop к ClickHouse Observability на Windows-хосте через опубликованный HTTP-порт `18127`. Если вы изменили учётные данные или порт в `.env`, укажите здесь актуальные значения. После сохранения конфигурации перезапустите Claude Desktop.

### Примеры промптов для ClickHouse MCP

После подключения MCP можно задавать вопросы на естественном языке. Например:

1. **Просмотр баз и таблиц**

   > List the databases available in ClickHouse, then show me the tables in one of them.
   >
2. **Оценка ошибок по `error_log`**

   > Посмотри таблицу `observability.error_log` и оцени состояние кластера. Сколько ошибок возникает за единицу времени? Покажи динамику ошибок по времени, укажи наиболее частые ошибки и объясни, достаточно ли данных для вывода о состоянии кластера.
   >
3. **Общая оценка наблюдаемости и нагрузки**

   > Проанализируй таблицы в базе `observability` и оцени состояние кластера: какие запросы создают наибольшую нагрузку, есть ли ошибки и какова общая нагрузка? Приведи временной интервал и метрики, на которых основаны выводы.
   >
4. **Нагрузка от SELECT-запросов по узлам за последние пять минут**

   > За последние 5 минут оцени нагрузку от `SELECT`-запросов по узлам ClickHouse. Используй поле `hostname` для группировки, сравни количество запросов, суммарную и среднюю длительность, а также прочитанные строки и байты. Выдели наиболее нагруженный узел и объясни критерий оценки.
   >

Результаты зависят от того, какие данные успели собрать потоки NiFi. Для корректного сравнения просите MCP указывать временной интервал, используемые поля и критерий нагрузки; выводы по небольшому числу событий следует считать предварительными.

## Полезные команды

Посмотреть логи всех сервисов:

```powershell
docker compose logs -f
```

Посмотреть логи конкретного сервиса:

```powershell
docker compose logs -f nifi
docker compose logs -f clickhouse-observability
docker compose logs -f clickhouse-mcp
```

Перезапустить сервисы:

```powershell
docker compose restart
```

Остановить стек:

```powershell
docker compose down
```

Команда `docker compose down -v` также удаляет именованные Docker volumes, включая хранилища Open WebUI и NiFi. Используйте её только если действительно хотите удалить эти данные.

## Основные каталоги

- `docker-compose.yml` — конфигурация Docker Compose.
- `.env` — учётные данные и параметры сервисов.
- `clickhouse-cluster/` — конфигурация и локальные данные четырёх узлов кластера.
- `clickhouse-observability-storage/` — конфигурация, схема и локальные данные ClickHouse Observability.
- `clickhouse-mcp/` — Dockerfile MCP-сервиса.
- `nifi/flow/flow.json.gz` — файл flow, который нужно указать в `nifi.properties` после запуска NiFi.
