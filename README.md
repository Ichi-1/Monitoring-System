
[![docker](https://img.shields.io/badge/-Docker-464646?style=for-the-badge&logo=docker)](https://www.docker.com/)
[![Grafana](https://img.shields.io/badge/-Grafana-464646?style=for-the-badge&logo=Grafana)](https://grafana.com/)
[![Prometheus](https://img.shields.io/badge/-Prometheus-464646?style=for-the-badge&logo=Prometheus)](https://prometheus.io/docs/introduction/overview/)


# Навигация

- [Описание](#описание)
- [Переменные окружения](#переменные-окружения)
    - [Сбор логов](#сбор-логов)
        - [Promtail](#promtail)
            - [Экспортёр и агрегатор установлены на одной машине](#экспортёр-и-агрегатор-установлены-на-одной-машине)
            - [Экспортёт установлен на одном сервере а агрегатор на другом](#экспортёт-установлен-на-одном-сервере-а-агрегатор-на-другом)
    - [Сбор метрик: Prometheus и источники данных](#сбор-метрик-prometheus-и-источники-данных)
- [Примечания](#примечания)


# Описание

Система мониторинга базируется на следующих компонентах:
 - [Prometheus](https://prometheus.io/docs/introduction/overview/) - позволяет собирать метрики из различных источников и сохранять их в базу в виде временных рядов (time-series data model)
    - Источники (экспортеры) данных:
        - [cAdvisor](https://github.com/google/cadvisor) - сервис "из-коробки" собирает системные метрики с запущенных контейнеров
        - [Node Exporter](#https://github.com/prometheus/node_exporter) - сервис, собирающий системные метрики операционной системы хоста
        - [Список поддерживаемых экспортеров](https://prometheus.io/docs/instrumenting/exporters/) для ознакомления
 - [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/) - агент-экспортер, который собирает с контейнеров логи и отправляет в агрегатор Loki. <b>Принцип работы в 3 шага</b>:
    - Discovers targets
    - Attaches labels to log streams
    - Pushes them to the Loki instance.
 - [Loki](https://github.com/grafana/loki) - многокомпонентный агрегатор (читай база данных) логов. Обеспечивает приём, хранение и очитску логов а так же выполнение queyr-запросов для их получения.
 - [Grafana](https://grafana.com/) - предоставляет централизованный доступ к веб-интерфейсу, для наглядной визуализации и просмотра собранных данных.


# Переменные окружения

```
{
  "GF_INSTALL_PLUGINS": "grafana-oncall-app",
  "GF_SERVER_ROOT_URL": "%(protocol)s://%(domain)s/",
  "GF_AUTH_ANONYMOUS_ENABLED": "false",
  "GF_USERS_ALLOW_SIGN_UP": "false"
  "GF_SECURITY_ADMIN_PASSWORD": "",
  "GF_SECURITY_ADMIN_USER": "",
  "GF_SERVER_DOMAIN": "",
  "GF_SMTP_ENABLED": "true",
  "GF_SMTP_FROM_ADDRESS": "",
  "GF_SMTP_HOST": "",
  "GF_SMTP_PASSWORD": "",
  "GF_SMTP_USER": "",
}
```


# Сбор логов

## Promtail

Promtail является экспортёром логов и может быть установлен на каждой машине, где установлена служба docker и запущенно N контейнеров, логи с которых требуется собирать и просматривать. Конфигуация будет отличаться в следующих случаях.


## Экспортёр и агрегатор установлены на одной машине

В таком случае необходим запущенный агрегатор Loki с параметром:
```
services:
    loki:
        ...

        ports:
            expose: 
            - 3100
```
После запуска сервиса по адрессу http://loki:3100/loki/api/v1/push будет доступен эндопоинт для экспорта в него логов. [Подробнее об HTTP API Grafana Loki](https://grafana.com/docs/loki/latest/api/)

Конфигурация Promtail в docker-compose будет выглядеть следующим образом:
```
promtail:
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock 
      - ./promtail-config.yaml:/etc/promtail/promtail-config.yaml
    command: 
      - --config.file=/etc/promtail/promtail-config.yaml
    depends_on:
      - loki
```
Монтируем в контейнеров docker.sock хоста для корректной работы `docker service discovery` и пробрасываем в контейнер свой .yml конфиг для Promtail.

`promtail-config.yaml` будет выглядеть следующим образом:
```
client:
    url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker-container-logs
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 30s

    relabel_configs:
      - source_labels: ['__meta_docker_container_label_com_docker_compose_project']
        regex: '(.*)'
        target_label: '${PREFIX}_compose_project'
```
`PREFIX` может быть любым, и служит для идентификации логов с конкретного docker-хоста, задается в качестве переменной окружения. В качестве `client` указываем  loki:3100, т.к. и экспортёр и агрегатор находятся в сети одного докер-хоста

## Экспортёт установлен на одном сервере, а агрегатор на другом

В таком случае, необходимо cделать доступным для внешних подключений эндпоинт Loki:

```
services:
    loki:
        ...

        labels:
            - "traefik.enable=true"
            - "traefik.subdomain=loki"
            - "traefik.http.services.loki.loadbalancer.server.port=3100"
            - "traefik.http.routers.loki.tls.certresolver=letsEncrypt"
```

Запустить на удаленной машине Promtail, изменив конфигурацию в `promtail-config.yaml`:

```
client:
  url: https://your.domain/loki/api/v1/push

scrape_configs:
  - job_name: docker-container-logs
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 30s

    relabel_configs:
      - source_labels: ['__meta_docker_container_label_com_docker_compose_project']
        regex: '(.*)'
        target_label: '${PREFIX}_compose_project'

```
При такой конфигурации Promtail будет экспортировать собранные на удалённом сервере логи из docker и направлять их в указанный в `client` эндпоинт агрегатора


# Сбор метрик: Prometheus и источники данных

Prometheus активно собирает/пуллит метрики с таргетов и формирует базу данных временных рядов. В качестве таргетов для сбора метрик указаны `cAdvisor` и `node-exporter`:
```
scrape_configs:
  - job_name: prom
    scrape_interval: 30m
    static_configs:
    - targets: ['cadvisor:8080', 'node-exporter:9100']
```
