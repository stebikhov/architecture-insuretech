# Задание 2. Динамическое масштабирование контейнеров

Настоящая директория содержит решение обеих частей задания: динамическая
маршрутизация тестового приложения `scaletestapp` с применением Horizontal Pod
Autoscaler (HPA) на основании утилизации оперативной памяти (часть 1) и на
основании количества запросов в секунду — RPS (часть 2).

## Состав директории

| Файл | Назначение |
|---|---|
| `deployment.yaml` | Манифест Deployment тестового приложения (1 реплика, лимит памяти 30Mi, аннотации `prometheus.io/*` для сбора метрик). |
| `service.yaml` | Манифест Service типа ClusterIP для доступа к приложению по порту 8080. |
| `hpa-memory.yaml` | Часть 1. Манифест HPA по утилизации памяти (цель — 80%, диапазон реплик 1–10). |
| `hpa-rps.yaml` | Часть 2. Доработанный манифест HPA по количеству запросов в секунду (метрика `http_requests_per_second`, цель — 10 RPS на под, диапазон реплик 1–10). |
| `prometheus-values.yaml` | Значения Helm-чарта `prometheus-community/prometheus` (минимальная конфигурация: только сервер, без Alertmanager и Pushgateway). |
| `prometheus-adapter-values.yaml` | Значения Helm-чарта `prometheus-community/prometheus-adapter`: правило преобразования `http_requests_total` в метрику `http_requests_per_second` Custom Metrics API. |
| `locustfile.py` | Сценарий нагрузочного теста Locust, обращающийся к ресурсу `GET /`. |
| `scaling-evidence.log` | Часть 1. Вывод `kubectl get/describe hpa`, `kubectl get pods` и `kubectl get events` для масштабирования по памяти. |
| `scaling-evidence-rps.log` | Часть 2. Аналогичный вывод для масштабирования по RPS, включая перечень доступных кастомных метрик. |
| `dashboard-deployments.png` | Скриншот дашборда Kubernetes со страницей Deployment `scaletestapp` (2/2 реплики под нагрузкой). |
| `dashboard-hpa.png` | Скриншот дашборда Kubernetes со страницей HorizontalPodAutoscaler. |
| `prometheus-targets.png` | Часть 2. Скриншот Prometheus Web UI, раздел Targets: цель `kubernetes-pods` для `scaletestapp` в состоянии `UP`. |
| `prometheus-graph-rps.png` | Часть 2. Скриншот Prometheus Web UI, раздел Graph: результат запроса `sum(rate(http_requests_total[2m])) by (pod)`. |

## Примечание к образу приложения

В задании указан тег
`ghcr.io/yandex-practicum/scaletestapp:sha256-eff20ae3ae2d596375f9ed6d612a78d149a35a66cd2907ea90d7175ca918c993.sig`.
Этот тег ссылается на артефакт подписи cosign (`application/vnd.dev.cosign.simplesigning.v1+json`),
а не на образ контейнера, вследствие чего его непосредственная загрузка приводит к ошибке
`ErrImagePull`. В манифесте `deployment.yaml` использован образ по тому же digest
без суффикса `.sig`:

    ghcr.io/yandex-practicum/scaletestapp@sha256:eff20ae3ae2d596375f9ed6d612a78d149a35a66cd2907ea90d7175ca918c993

что соответствует тегу `latest` того же репозитория.

## Инструкция по запуску

### 1. Поднятие локального кластера и включение metrics-server

    minikube start --driver=docker --memory=4096 --cpus=2
    minikube addons enable metrics-server

Проверка готовности узла и системных подов:

    kubectl get nodes
    kubectl get pods -n kube-system

Под `metrics-server-*` должен перейти в состояние `1/1 Running`.

### 2. Применение манифестов Deployment, Service и HPA

Если используемый локальный движок Docker не имеет прямого доступа к образу
по тегу `.sig` (см. примечание выше), образ предварительно загружается в узел
Minikube по digest:

    docker pull ghcr.io/yandex-practicum/scaletestapp@sha256:eff20ae3ae2d596375f9ed6d612a78d149a35a66cd2907ea90d7175ca918c993
    minikube image load ghcr.io/yandex-practicum/scaletestapp@sha256:eff20ae3ae2d596375f9ed6d612a78d149a35a66cd2907ea90d7175ca918c993

Применение манифестов:

    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml
    kubectl apply -f hpa-memory.yaml

Проверка состояния объектов:

    kubectl get deployment scaletestapp
    kubectl get svc scaletestapp
    kubectl get hpa scaletestapp-hpa

Столбец `TARGETS` в выводе `kubectl get hpa` отображает текущую утилизацию памяти
относительно целевого значения 80%. Значение `<unknown>` в этом столбце в первые
секунды после создания объекта свидетельствует о том, что metrics-server ещё не
успел собрать первую метрику, и не является ошибкой конфигурации.

### 3. Проверка доступности сервиса

Порт приложения проброшен на локальную машину:

    kubectl port-forward svc/scaletestapp 8080:8080

Проверка ресурсов приложения в отдельном терминале:

    curl http://localhost:8080/
    curl http://localhost:8080/metrics

Первый запрос возвращает идентификатор пода, второй — метрики в формате Prometheus,
включая счётчик `http_requests_total`.

### 4. Генерация нагрузки с помощью Locust

Установка Locust (при необходимости):

    pip install locust

Запуск нагрузочного теста в headless-режиме из директории `Task2`:

    python -m locust -f locustfile.py --host=http://localhost:8080 --headless -u 1000 -r 50 --run-time 8m

Параметры `-u` (количество пользователей) и `-r` (hatch rate) подобраны с учётом
низкого лимита памяти контейнера (30Mi): при указанных значениях утилизация
памяти превышает целевой порог 80% в течение первых 5–7 минут теста.
Также доступен интерактивный режим без флага `--headless`:
веб-интерфейс открывается по адресу `http://localhost:8089`.

### 5. Наблюдение за изменением количества реплик

Отслеживание в реальном времени:

    kubectl get hpa scaletestapp-hpa --watch
    kubectl get pods -l app=scaletestapp --watch

Просмотр дашборда Kubernetes:

    minikube dashboard

Хронология масштабирования и связанных событий:

    kubectl describe hpa scaletestapp-hpa
    kubectl get events --sort-by='.lastTimestamp'

## Результат проверки

В файле `scaling-evidence.log` зафиксирован полный цикл автоматического
масштабирования, инициированного нагрузочным тестом:

1. При запуске под приложения потреблял значительно меньше 80% лимита памяти
   (30Mi), количество реплик было равно 1.
2. После генерации нагрузки Locust (1000 пользователей) утилизация памяти
   превысила целевой порог, что подтверждается событием HPA:
   `SuccessfulRescale ... New size: 2; reason: memory resource utilization
   (percentage of request) above target`.
3. В процессе роста нагрузки исходный под превысил лимит памяти 30Mi и был
   перезапущен со статусом `OOMKilled` — это дополнительно подтверждает, что
   лимит памяти контейнера установлен корректно и действительно ограничивает
   потребление ресурсов.
4. После остановки нагрузочного теста и снижения утилизации памяти HPA
   автоматически уменьшил количество реплик обратно до 1, что подтверждается
   событием `SuccessfulRescale ... New size: 1; reason: All metrics below target`.

Таким образом, конфигурация HPA на основании утилизации оперативной памяти
обеспечивает автоматическое увеличение количества реплик приложения при
возрастании нагрузки и последующее сокращение реплик после её снижения.

# Часть 2. Динамическая маршрутизация на основании количества запросов в секунду

Часть 2 расширяет конфигурацию из части 1 внешней системой мониторинга
Prometheus, из которой Kubernetes получает кастомную метрику количества
запросов в секунду (RPS) на один под приложения. Deployment и Service
из части 1 используются без изменений; заменяется только манифест HPA.

## Архитектура решения

1. Приложение `scaletestapp` уже предоставляет метрики в формате Prometheus
   по пути `/metrics` (эндпоинт задан образом приложения, см. `Task.txt`).
2. Poды приложения размечены аннотациями `prometheus.io/scrape: "true"`,
   `prometheus.io/port: "8080"`, `prometheus.io/path: "/metrics"`
   (файл `deployment.yaml`) — Prometheus обнаруживает их автоматически через
   стандартный job `kubernetes-pods` из своей конфигурации автообнаружения.
3. Prometheus Server (Helm-чарт `prometheus-community/prometheus`) собирает
   счётчик `http_requests_total` с каждого пода.
4. Prometheus Adapter (Helm-чарт `prometheus-community/prometheus-adapter`)
   транслирует запрос PromQL `sum(rate(http_requests_total{...}[2m])) by (pod)`
   в метрику `http_requests_per_second`, публикуемую через Kubernetes
   Custom Metrics API (`custom.metrics.k8s.io/v1beta1`) в разрезе "на под".
5. HPA (`hpa-rps.yaml`) типа `Pods` считывает метрику `http_requests_per_second`
   через Custom Metrics API и сравнивает её среднее значение по подам
   с целью `averageValue: "10"` (10 запросов в секунду на под).

## Инструкция по запуску

Пункты 1–3 из инструкции части 1 (поднятие кластера, включение
metrics-server, применение `deployment.yaml` и `service.yaml`) выполняются
без изменений и не повторяются.

### 1. Установка Prometheus

Добавление репозитория Helm-чартов (если не выполнено ранее):

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update prometheus-community

Создание отдельного пространства имён и установка Prometheus с минимальной
конфигурацией (без Alertmanager и Pushgateway, без персистентного хранилища —
для учебного стенда используется хранение метрик только в памяти сервера):

    kubectl create namespace monitoring
    helm install prometheus prometheus-community/prometheus -n monitoring -f prometheus-values.yaml

Проверка готовности:

    kubectl get pods -n monitoring

Под `prometheus-server-*` должен перейти в состояние `2/2 Running`
(контейнеры `prometheus-server` и `configmap-reload`).

### 2. Настройка экспорта метрик приложения в Prometheus

Экспорт уже настроен через аннотации `prometheus.io/*` в `deployment.yaml`
(см. раздел «Архитектура решения»). Применение обновлённого манифеста:

    kubectl apply -f deployment.yaml

Дополнительная настройка Prometheus для сбора метрик по аннотациям не
требуется: соответствующий job `kubernetes-pods` присутствует в конфигурации
чарта по умолчанию.

### 3. Проверка поступления метрик в Prometheus (Prometheus Web UI)

Проброс порта Prometheus на локальную машину:

    kubectl port-forward -n monitoring svc/prometheus-server 9090:80

Раздел **Targets** (`http://localhost:9090/targets`): в группе задания
`kubernetes-pods` должна отображаться цель с меткой `pod=scaletestapp-...`
в состоянии `UP` — зафиксировано на скриншоте `prometheus-targets.png`.

Раздел **Graph** (`http://localhost:9090/graph`): запрос

    sum(rate(http_requests_total[2m])) by (pod)

возвращает значение, отличное от нуля, при наличии активной нагрузки —
зафиксировано на скриншоте `prometheus-graph-rps.png`.

Проверка тем же способом через `curl` (без браузера):

    curl -s "http://localhost:9090/api/v1/query?query=http_requests_total"

### 4. Установка Prometheus Adapter и настройка Custom Metrics API

    helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring -f prometheus-adapter-values.yaml

Проверка регистрации Custom Metrics API и готовности адаптера:

    kubectl get pods -n monitoring
    kubectl get apiservice v1beta1.custom.metrics.k8s.io

Столбец `AVAILABLE` в выводе `kubectl get apiservice` должен принять значение
`True`. Проверка перечня и значения кастомной метрики:

    kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"
    kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_per_second"

### 5. Применение HPA по RPS

    kubectl apply -f hpa-rps.yaml

Этой командой объект `scaletestapp-hpa`, созданный в части 1 на основании
памяти, заменяется новой версией на основании метрики
`http_requests_per_second`. Проверка:

    kubectl get hpa scaletestapp-hpa
    kubectl describe hpa scaletestapp-hpa

Столбец `TARGETS` отображает текущее среднее значение RPS на под относительно
целевого значения `10`.

### 6. Генерация нагрузки и наблюдение за масштабированием

Проброс порта приложения (если ещё не выполнен):

    kubectl port-forward svc/scaletestapp 8080:8080

Запуск нагрузочного теста аналогично части 1:

    python -m locust -f locustfile.py --host=http://localhost:8080 --headless -u 50 -r 10 --run-time 3m

При количестве одновременных пользователей 50 и интервале между запросами
1–5 секунд суммарная интенсивность запросов превышает целевое значение
10 RPS на под, что вызывает срабатывание HPA в течение первой минуты теста.

Наблюдение за изменением количества реплик:

    kubectl get hpa scaletestapp-hpa --watch
    kubectl get pods -l app=scaletestapp --watch
    kubectl describe hpa scaletestapp-hpa
    kubectl get events --sort-by='.lastTimestamp'

Просмотр дашборда Kubernetes: `minikube dashboard`.

## Результат проверки

В файле `scaling-evidence-rps.log` зафиксирован полный цикл автоматического
масштабирования по метрике RPS:

1. В состоянии покоя (без нагрузки) HPA показывал `0/10` по метрике
   `http_requests_per_second`, количество реплик было равно 1.
2. После генерации нагрузки Locust (50 пользователей) среднее значение RPS
   на под превысило целевой порог 10, что подтверждается событием HPA:
   `SuccessfulRescale ... New size: 2; reason: pods metric
   http_requests_per_second above target`.
3. После остановки нагрузочного теста и снижения RPS ниже целевого значения
   HPA автоматически уменьшил количество реплик обратно до 1 (по истечении
   стандартного окна стабилизации scale-down), что подтверждается событием
   `SuccessfulRescale ... New size: 1; reason: All metrics below target`.

Дополнительно скриншоты `prometheus-targets.png` и `prometheus-graph-rps.png`
подтверждают, что метрики приложения `scaletestapp` (в частности,
`http_requests_total`) корректно собираются Prometheus и доступны для
дальнейшей агрегации в метрику RPS.

Таким образом, конфигурация HPA на основании внешней метрики количества
запросов в секунду, полученной через Prometheus Adapter и Custom Metrics API,
обеспечивает автоматическое масштабирование приложения по признаку,
не ограниченному стандартными метриками CPU и памяти.
