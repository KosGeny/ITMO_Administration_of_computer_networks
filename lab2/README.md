# Лабораторная работа 2. Loki + Zabbix + Grafana. Команда 10.

## Часть 1. Логирование
Создаём файл docker-compose.yml, содержащий тестовый сервис Nextcloud, Loki, Promtail, Grafana, Zabbix и Postgres для него.

```yml
services:
  nextcloud:
    image: nextcloud:29.0.6
    container_name: nextcloud
    ports:
      - "8080:80"
    volumes:
      - nc-data:/var/www/html/data

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    volumes:
      - nc-data:/opt/nc_data
      - ./promtail_config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:11.2.0
    container_name: grafana
    environment:
      GF_PATHS_PROVISIONING: /etc/grafana/provisioning
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    command: /run.sh

  postgres-zabbix:
    image: postgres:15
    container_name: postgres-zabbix
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
    volumes:
      - zabbix-db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "zabbix"]
      interval: 10s
      retries: 5
      start_period: 5s

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-6.4-latest
    container_name: zabbix-back
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      DB_SERVER_HOST: postgres-zabbix
    ports:
      - "10051:10051"
    depends_on:
      - postgres-zabbix

  zabbix-web-nginx-pgsql:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-6.4-latest
    container_name: zabbix-front
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      DB_SERVER_HOST: postgres-zabbix
      ZBX_SERVER_HOST: zabbix-back
    ports:
      - "8082:8080"
    depends_on:
      - postgres-zabbix

volumes:
  nc-data:
  grafana-data:
  zabbix-db:
```

Создаём файл `promtail_config.yml`. Это конфигурационный файл для Promtail, в котором описано, какой слушать порт, какие логи собирать и куда их отправлять.

```yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
    - localhost
    labels:
      job: nextcloud_logs
      __path__: /opt/nc_data/*.log
```

Запускаем docker-compose файл.

![docker-compose_done](assets/docker-compose_done.png)

![docker-compose_containers_work](assets/docker-compose_containers_work.png)

Видим, что все контейнеры успешно собраны и запущены, на это указывает статус Up.

Далее инициализируем Nextcloud. Заходим на веб-интерфейс через внешний порт `8080`, который мы указали в docker-compose файле. Создаём учётную запись админа.

![nextcloud_create_admin](assets/nextcloud_create_admin.png)

Проверяем, что логи пишутся в нужный нам файл `/var/www/html/data/nextcloud.log`.

![nextcloud_logs](assets/nextcloud_logs.png)

После инициализации Nextcloud проверяем в логах Promtail, что он подхватил нужный лог-файл.

![promtail_logs](assets/promtail_logs.png)

## Часть 2. Мониторинг

Настраиваем Zabbix. Подключаемся к `http://localhost:8082` и вводим логин `Admin` и пароль `zabbix`.

В разделе `Data collection -> Templates` импортируем шаблон `template.yml` для мониторинга Nextcloud.

```yml
zabbix_export:
    version: '6.4'
    template_groups:
      - uuid: a571c0d144b14fd4a87a9d9b2aa9fcd6
        name: Templates/Applications
    templates:
      - uuid: a615dc391a474a9fb24bee9f0ae57e9e
        template: 'Test ping template'
        name: 'Test ping template'
        groups:
          - name: Templates/Applications
        items:
          - uuid: a987740f59d54b57a9201f2bc2dae8dc
            name: 'Nextcloud: ping service'
            type: HTTP_AGENT
            key: nextcloud.ping
            value_type: TEXT
            trends: '0'
            preprocessing:
              - type: JSONPATH
                parameters:
                  - $.body.maintenance
              - type: STR_REPLACE
                parameters:
                  - 'false'
                  - healthy
              - type: STR_REPLACE
                parameters:
                  - 'true'
                  - unhealthy
            url: 'http://{HOST.HOST}/status.php'
            output_format: JSON
            triggers:
              - uuid: a904f3e66ca042a3a455bcf1c2fc5c8e
                expression: 'last(/Test ping template/nextcloud.ping)="unhealthy"'
                recovery_mode: RECOVERY_EXPRESSION
                recovery_expression: 'last(/Test ping template/nextcloud.ping)="healthy"'
                name: 'Nextcloud is in maintenance mode'
                priority: DISASTER
```

![zabbix_import_template-1](assets/zabbix_import_template-1.png)
![zabbix_import_template-2](assets/zabbix_import_template-2.png)
![zabbix_import_template-3](assets/zabbix_import_template-3.png)

Видим, что шаблон успешно загружен.

Сделаем так, чтобы Zabbix мог обращаться к Nextcloud по внутреннему DNS-имени `nextcloud` в docker-сети. Добавляем `nextcloud` в список `trusted_domains`.

![nextcloud_trusted_domains](assets/nextcloud_trusted_domains.png)

В разделе `Data collections -> Hosts` добавляем хост. Указываем имя контейнера `nextcloud`, видимое имя `Nextcloud Server`, хост группу `Applications`. Подключаем добавленный ранее шаблон мониторинга `Templates/Applications -> Test ping template`.

![zabbix_new_host](assets/zabbix_new_host.png)

![zabbix_new_host_successful](assets/zabbix_new_host_successful.png)

Видим, что хост успешно создан, и он будет мониториться.

Перейдём в раздел `Monitoring -> Latest data` для просмотра логов.

![zabbix_latest_data](assets/zabbix_latest_data.png)

Видим, что данные появились.

Теперь попробуем включить `maintenance mode`, чтобы посмотреть, как отреагирет система.

![nextcloud_maintenance_mode_on](assets/nextcloud_maintenance_mode_on.png)


![zabbix_nextcloud_problem](assets/zabbix_nextcloud_problem.png)
Ловим проблему.

Теперь выключаем `maintenance mode`, чтобы посмотреть, решилась ли проблема.

![nextcloud_maintenance_mode_off](assets/nextcloud_maintenance_mode_off.png)


![zabbix_nextcloud_problem_resolved](assets/zabbix_nextcloud_problem_resolved.png)

Всё успешно.

## Часть 3. Визуализация.

В контейнере Grafana устанавливаем плагин `alexanderzobnin-zabbix-app` для интеграции Grafana с Zabbix и перезапускаем контейнер.

![grafana_alexanderzobnin-zabbix-app_plugin](assets/grafana_alexanderzobnin-zabbix-app_plugin.png)

Теперь заходим в Grafana по адресу `http://localhost:3000`, переходим в раздел `Administration -> Plugins`, находим Zabbix и активируем его.

![grafana_zabbix_enable](assets/grafana_zabbix_enable.png)

Далее переходим в раздел `Connections -> data sources -> Loki`, чтобы подключить Loki к Grafana. В настройках подключения указываем имя `loki_datasource` и адрес `http://loki:3100`, все остальные настройки оставляем по умолчанию.

![](assets/grafana_loki_datasource-1.png)

Сохраняем подключение, нажав Save & test.

![](assets/grafane_loki_datasource-2.png)

Ошибок нет, значит, в первой части мы сделали всё правильно.

То же самое делаем с Zabbix. Далее переходим в раздел `Connections -> data sources -> Zabbix`. В настройках подключения указываем имя `zabbix_datasource` и адрес `http://zabbix-front:8080/api_jsonrpc.php`, заполняем имя пользователя `Admin` и пароль `zabbix`.

![grafana_zabbix_datasource-1](assets/grafana_zabbix_datasource-1.png)

Сохраняем подключение, нажав Save & test.

![grafana_zabbix_datasource-2](assets/grafana_zabbix_datasource-2.png)

Ошибок нет, подключение успешно.

Переходим в раздел `Explore` и выбираем в качестве источника `loki_datasource` и индекс `job`. Ставим фильтр на последние 6 часов и нажимаем `Run query`.

![grafana_loki_datasource_logs](assets/grafana_loki_datasource_logs.png)

Видим нащи логи.

Аналогично выбираем в качестве источника `zabbix_datasource` и выставляем фильтры: тип запроса - `Text`, группа - `Applications`, хост - `Nextcloud Server`, элемент данных (item) `Nextcloud: ping service`. Оставляем последние 6 часов.

![grafana_zabbix_datasource_logs](assets/grafana_zabbix_datasource_logs.png)


## Дополнительные задания
### "Поиграться" с запросами

Обратимся к loki_datasource и найдёми логи, содержащие `Failed`.

![extra_task_1-1](assets/extra_task_1-1.png)

Обратимся к zabbix_datasource и установим text filter `unhealthy`.

![extra_task_1-2](assets/extra_task_1-2.png)

### Создать дашборды в Grafana

Создадим таблицу логов с loki_datasource.

![grafana_nextcloud_loki_logs](assets/grafana_nextcloud_loki_logs.png)

Создадим stat с текущим состоянием системы (healthy/unhealthy), взяв данные из zabbix_datasource.

![grafana_nextcloud_zabbix_healthy](assets/grafana_nextcloud_zabbix_healthy.png)

Объединим их в один дашборд.

![grafana_nextcloud_dashboard-1](assets/grafana_nextcloud_dashboard-1.png)

Вместо stat можно использовать state timeline. Тогда будет видно, как изменялось сотстояние системы со временем в рассматриваемый промежуток.

![grafana_nextcloud_zabbix_healthy_timeline](assets/grafana_nextcloud_zabbix_healthy_timeline.png)

Посмотрим на единый дашборд.

![grafana_nextcloud_dashboard-2](assets/grafana_nextcloud_dashboard-2.png)


## Ответы на вопросы
### Чем SLO отличается от SLA?
SLO (Service Level Objective) - внутренняя цель по уровню качества, которую ставит перед собой компания/команда. Например, достичь аптайма в 99.9%

SLA (Service Level Agreement) - это договорные обязательства поставщика услуги перед клиентом. В нём прописано, какой уровень услуг гарантирован, и что произойдёт, если эти условия нарушаются. 

### Чем отличается инкрементальный бэкап от дифференциального?
Инкрементальный бэкап сохраняет только те данные, которые изменились с момента последнего бэкапа любого типа. Для восстановления данных из инкрементной копии необходим доступ ко всем предыдущим бэкапам в цепочке. Такой подход позволяет ускорить процесс создания бэкапов, сэкономить дисковое пространство и снизить нагрузку на сеть, однако процесс восстановления труднее.  

Дифференциальный бэкап сохраняет все изменения с момента последнего полного бэкапа, из-за чего он тяжелее и процесс создания бэкапов медленее. Сначала создаётся полная копия данных, а затем копируются только изменения, произошедшие с момента создания этой полной копии. Процесс восстановления проще, нужны лишь полная копия и последняя дифференциальная, что ускоряет процесс по сравнению с инкрементным подходом.

### В чём разница между мониторингом и observability?
Мониторинг предполагает сбор заранее определённых метрик, логов и алертов для выявления известных проблем и отслеживания состояния системы.  

Observability позволяет глубоко анализировать причины сбоев, даже если конкретная проблема не была заранее предсказана, в сложных распределенных системах через корреляцию данных. Основой observability являются три ключевых компонента: метрики, логи и трейсы.
