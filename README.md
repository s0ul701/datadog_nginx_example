# Настройка связки DataDog + Nginx

[DataDog](https://www.datadoghq.com/) - система мониторинга состояния сервера, обладающая широким спектром возможностей.
Данная система является клиент-серверной, т.е. на контролируемом сервере устанавливается ТОЛЬКО агент, который отсылает метрики на сервера DataDog.

---

## Подготовка

1. [Регистрация](https://app.datadoghq.com/signup) (создание личного кабинета на стороне DataDog, откуда и осуществляется мониторинг)
2. [Получение API-ключа](https://app.datadoghq.eu/account/settings#api)

---

## 1. Подключение Docker-интеграции

Данная интеграция позволяет мониторить состояние запущенных на сервере контейнеров (на достаточно высоком уровне абстракции: CPU, RAM, I/O и т.д.), не анализируя специфичные для запущенных в контейнерах приложений метрики.

***./docker-compose.yml:***

```yaml
version: '3.3'
services:
    ...other services...

    datadog-agent:
        image: datadog/agent:7.26.0-jmx
        env_file:
            - ./datadog/.env
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock:ro  # с помощью вольюмов осуществляется
            - /proc/:/host/proc/:ro                         # сбор метрик с контейнеров/сервера
            - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro

    ...more other services...
```

***./datadog.env:***

```yaml
DD_API_KEY=<YOUR_DATADOG_API_KEY>
DD_SITE=<YOUR_DATADOG_DOMEN>

DD_PROCESS_AGENT_ENABLED=true   # позволяет просматривать процессы сервера/контейнеров в DataDog
```

Результаты настроек доступны по [ссылке](https://app.datadoghq.eu/containers):

*Ссылки*:

1. [Документация](https://docs.datadoghq.com/integrations/faq/compose-and-the-datadog-agent/) по базовой настройке связки DataDog/Docker/Docker Compose;
2. Базовая [документация](https://docs.datadoghq.com/agent/docker/?tab=standard) по Datadog Agent.

<br>

## 2. Подключение Nginx-интеграции

Данная интеграция позволяет отслеживать специфичные для Nginx метрики (количество активных соединений, количество запросов в секунду и т.д.) и его логи.

***./docker-compose.yml:***

```yaml
version: '3.3'
services:
    ...other services...

    nginx:
        build:                          # кастомный образ необходим для задания
            context: ./nginx            # Nginx кастомного файла настроек
            dockerfile: ./Dockerfile
        ports:
            - 80:80
            - 81:81     # порт Nginx-сервера, с которого DataDog получает метрики
        volumes:
            - ./nginx_logs:/nginx_logs  # вольюм для хранения папки (файла) с логами
        labels:
            com.datadoghq.ad.check_names: '["nginx"]'    # на основе этого лейбла DataDog определяет, какое приложение работает в контейнере (не менять!)
            com.datadoghq.ad.init_configs: '[{}]'   # инициализирующие настройки для взаимодействия DataDog и Nginx (не менять!)
            com.datadoghq.ad.instances: >-  # основной блок настройки соединения DataDog и Nginx
                [{
                    "nginx_status_url": "http://%%host%%:81/nginx_status/",     # вместо этой шаблонной переменной DataDog подставляет IP-адрес контейнера с Nginx
                    "log_requests": "true"
                }]
            com.datadoghq.ad.logs: >-
                [
                    {
                        "type": "file", # тип источника логов
                        "source": "nginx", # название интеграции (не менять!)
                        "service": "nginx",  # имя сервиса для отображение в UI DataDog
                        "path": "/nginx_logs/access.log",  # путь до файла с логами (внутри контейнера DataDog-агента!)
                        "log_processing_rules": [{  # блок правил обработки логов
                            "type": "multi_line",   # сообщение DataDog`у о том, что логи могут быть многострочными
                            "name": "access_log",
                            "pattern" : "[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}" # паттерн начала унарного лог-сообщения
                        }]
                    },
                    {
                        "type": "file",
                        "source": "nginx",
                        "service": "nginx",
                        "path": "/nginx_logs/error.log",
                        "log_processing_rules": [{
                            "type": "multi_line",
                            "name": "error_log",
                            "pattern": "[0-9]{4}/[0-9]{2}/[0-9]{2}"
                        }]
                    }
                ]

    datadog-agent:
        image: datadog/agent:7.26.0-jmx
        env_file:
            - ./datadog.env
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock:ro
            - /proc/:/host/proc/:ro
            - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
            - /opt/datadog-agent/run:/opt/datadog-agent/run:rw  # вольюм позволяет сохранять логи локально на случай непредвиденных ситуаций
            - ./nginx_logs:/nginx_logs    # вольюм прокидывает логи Nginx-контейнера в DataDog-контейнер

    ...more other services...
```

***./nginx/Dockerfile:***

```Dockerfile
FROM nginx:1.19.10

RUN mkdir /nginx_logs          # создание директории

COPY ./nginx.conf /etc/nginx/nginx.conf     # копирование кастомного конфиг-файла для Nginx
```

***./nginx/nginx.conf:***

```nginx

user nginx;
worker_processes auto;

error_log /nginx_logs/error.log info;

events {
    worker_connections 1024;
}

http {
    log_format custom_log_format '$remote_addr [$time_local] '
                                 '"$request" $status $body_bytes_sent $request_time '
                                 '"$http_referer" "$http_user_agent"';
    access_log /nginx_logs/access.log custom_log_format;

    server {                        # данный виртуальный сервер отвечает
        listen 81;                  # за сбор метрик с Nginx DataDog-агентом
        server_name _;

        access_log off;

        location /nginx_status {
            stub_status;
            server_tokens on;
        }
    }

    server {
        listen 80;
        server_name _;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
}

```

***./datadog.env:***

```yaml
DD_API_KEY=<DATADOG_API_KEY>
DD_SITE=<DATADOG_DOMEN>

DD_PROCESS_AGENT_ENABLED=true   # позволяет DataDog-агенту просматривать процессы сервера/контейнеров в DataDog
DD_LOGS_ENABLED=true    # позволяет DataDog-агенту собирать логи с сервера/контейнеров
DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true   # включает у DataDog-агента сбор логов со всех контейнеров
```

./test_scripts/load_for_nginx.sh -- скрипт для создания тестовой нагрузки на Nginx для проверки корректности произведенных настроек.

Ссылки:

1. Базовая [документация](https://docs.datadoghq.com/integrations/nginx/?tab=containerizedr) по настройке Nginx-интеграции в Docker;
2. [Статья](https://docs.datadoghq.com/agent/docker/log/?tab=dockercompose) по настройке логирования DataDog-агентом;
3. [Статья](https://docs.datadoghq.com/agent/docker/integrations/?tab=docker) по настройке автообнаружения интеграций DataDog-агентом;
4. [Параметры](https://github.com/DataDog/integrations-core/blob/master/nginx/datadog_checks/nginx/data/conf.yaml.example) настройки Nginx в Datadog;
5. [Документация](https://docs.datadoghq.com/agent/faq/template_variables/) по шаблонным переменным.
