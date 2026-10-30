# Архитектура программного комплекса InsureTech

Проектирования и оптимизации целевой архитектуры программного комплекса компании InsureTech. Сервис функционирует в качестве страхового агрегатора и предоставляет услуги по подбору и оформлению договоров страхования как физическим лицам посредством веб-приложения, так и корпоративным клиентам через интерфейс прикладного программирования (Application Programming Interface, API).

Цель работы заключается в устранении архитектурных ограничений текущего решения (As-Is), обеспечении показателей доступности 99,9%, соблюдении целевых метрик отказоустойчивости (Recovery Time Objective, RTO — 45 минут; Recovery Point Objective, RPO — 15 минут), внедрении механизмов динамического масштабирования и разработке архитектурных решений для масштабирования бизнеса.

## Структура репозитория и артефакты заданий

В соответствии с требованиями к выполнению проекта репозиторий структурирован по директориям, соответствующим этапам проектной работы (`Task1`–`Task6`).

### Задание 1. Проектирование технологической архитектуры

Спроектирована целевая технологическая архитектура (To-Be), обеспечивающая геораспределенную отказоустойчивость, балансировку трафика между зонами доступности и репликацию базы данных PostgreSQL с обеспечением заданных RTO и RPO.

- Целевая схема технологической архитектуры в нотации C4 (модель контейнеров draw.io): [Task1/InureTech_технологическая архитектура_to-be.drawio](Task1/InureTech_технологическая%20архитектура_to-be.drawio)

### Задание 2. Динамическое масштабирование контейнеров

Реализовано и верифицировано горизонтальное автомасштабирование подов (Horizontal Pod Autoscaler, HPA) в кластере Kubernetes для тестового микросервиса `scaletestapp`.

**Часть 1. Автомасштабирование на основании утилизации оперативной памяти:**

- Манифест развертывания микросервиса: [Task2/deployment.yaml](Task2/deployment.yaml)
- Манифест сетевого сервиса: [Task2/service.yaml](Task2/service.yaml)
- Манифест HPA по утилизации памяти (целевой порог 80%): [Task2/hpa-memory.yaml](Task2/hpa-memory.yaml)
- Сценарий генерации нагрузки Locust: [Task2/locustfile.py](Task2/locustfile.py)
- Журнал регистрации масштабирования под нагрузкой: [Task2/scaling-evidence.log](Task2/scaling-evidence.log)
- Графические свидетельства масштабирования в дашборде Kubernetes:
  - [Task2/dashboard-deployments.png](Task2/dashboard-deployments.png) — состояние Deployment под нагрузкой
  - [Task2/dashboard-hpa.png](Task2/dashboard-hpa.png) — состояние объекта HorizontalPodAutoscaler
  - [Task2/kubernetus-deployment.png](Task2/kubernetus-deployment.png) — фиксация изменения числа реплик
  - [Task2/kubernetus-scale.png](Task2/kubernetus-scale.png) — график масштабирования подов

**Часть 2. Автомасштабирование на основании количества запросов в секунду (RPS):**

- Доработанный манифест HPA по кастомной метрике RPS: [Task2/hpa-rps.yaml](Task2/hpa-rps.yaml)
- Конфигурация Prometheus Server: [Task2/prometheus-values.yaml](Task2/prometheus-values.yaml)
- Конфигурация Prometheus Adapter для Custom Metrics API: [Task2/prometheus-adapter-values.yaml](Task2/prometheus-adapter-values.yaml)
- Скриншоты веб-интерфейса Prometheus:
  - [Task2/prometheus-targets.png](Task2/prometheus-targets.png) — сбор метрик с подов (раздел Targets)
  - [Task2/prometheus-graph-rps.png](Task2/prometheus-graph-rps.png) — график интенсивности запросов (раздел Graph)
- Журнал подтверждения масштабирования по RPS: [Task2/scaling-evidence-rps.log](Task2/scaling-evidence-rps.log)
- Подробная техническая документация и протокол испытаний: [Task2/README.md](Task2/README.md)

### Задание 3. Переход на событийно-ориентированную архитектуру (Event-Driven Architecture)

Решения по устранению узких мест синхронной агрегации и периодического опроса (polling) за счет перехода на асинхронный обмен сообщениями через брокер Apache Kafka с применением паттерна Transactional Outbox.

- Описание с перечнем проблем и архитектурных рисков: [Task3/problems_and_risks.md](Task3/problems_and_risks.md)
- Обновленная диаграмма контейнеров InsureTech в draw.io: [Task3/InureTech_технологическая архитектура_to-be.drawio](Task3/InureTech_технологическая%20архитектура_to-be.drawio)

### Задание 4. Проектирование процесса онлайн-продажи ОСАГО

Спроектирован отдельный микросервис `osago-aggregator` с распределенным состоянием в Redis, асинхронной интеграцией с ядром системы (`core-app`) через Apache Kafka, передачей котировок в веб-клиент по протоколу Server-Sent Events (SSE) и применением паттернов отказоустойчивости (Circuit Breaker, Timeout, Retry, Rate Limiting).

- Архитектурная схема в нотации C4 (draw.io): [Task4/InureTech_технологическая архитектура_to-be.drawio](Task4/InureTech_технологическая%20архитектура_to-be.drawio)
- Архитектурное описание проектных решений и паттернов отказоустойчивости: [Task4/osago_architecture.md](Task4/osago_architecture.md)

### Задание 5. Проектирование GraphQL API

Перевод контракта сервиса управления клиентскими данными (`client-info`) с архитектурного стиля REST на GraphQL с целью устранения проблем избыточной и недостаточной выборки данных (over-fetching и under-fetching).

- Спецификация схемы GraphQL: [Task5/client-info.graphql](Task5/client-info.graphql)

### Задание 6. Конфигурация ограничения интенсивности запросов (Rate Limiting)

- Конфигурационный файл Nginx: [Task6/nginx.conf](Task6/nginx.conf)

## Используемое программное обеспечение и инструментарий

- Среда оркестрации контейнеров: Minikube 1.38.1, Kubectl 1.36.2
- Мониторинг и сбор метрик: Prometheus, Prometheus Adapter
- Нагрузочное тестирование: Python 3.11.3, Locust 2.46.6
- Обратный прокси-сервер: Nginx
- Графическое моделирование архитектуры: Draw.io (нотация C4, библиотеки пиктограмм Yandex Cloud)
