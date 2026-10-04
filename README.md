# GAP-1: WordPress Stack Monitoring

## Stack
- Ubuntu 22.04
- Nginx + PHP-FPM + MySQL (WordPress)
- Prometheus + Alertmanager + Blackbox Exporter

## Exporters
- node_exporter :9100
- nginx-exporter :9113
- php-fpm-exporter :9253
- mysqld_exporter :9104
- blackbox_exporter :9115

## Prometheus
- scrape_interval: 5s
- UI: http://10.0.2.15:9090

## Alertmanager
- UI: http://10.0.2.15:9093

## Verification
- /targets all UP
- probe_success{job="blackbox"} = 1

## Architecture
Prometheus :9090 -> scrapes -> exporters
                             |- node_exporter :9100
                             |- nginx-exporter :9113
                             |- php-fpm-exporter :9253
                             |- mysqld_exporter :9104
                             `- blackbox_exporter :9115
Alertmanager :9093 <- receives alerts from Prometheus

## Verification
- systemctl is-active prometheus alertmanager blackbox_exporter
- ss -tlnp | grep -E ':(9090|9093|9115)'
- curl -s http://localhost:9090/-/healthy
- curl -s http://localhost:9093/-/healthy
- curl -s http://localhost:9115/-/healthy



GAP-1 #2: Prometheus + Alertmanager + Blackbox + VictoriaMetrics

## Что сделано :
- VictoriaMetrics установлена как долговременное хранилище метрик
- Retention: 14 дней
- Prometheus пишет метрики в VictoriaMetrics через remote_write
- Адрес записи: http://localhost:8428/api/v1/write
- Все метрики получают лейбл site: prod

## Файлы в репозитории:
- prometheus.yml — конфиг Prometheus (scrape + remote_write + external_labels)
- victoriametrics.service — systemd-юнит с retention 14d
- alertmanager.yml — конфиг Alertmanager
- alerts.yml — правила алертов
- blackbox.yml — конфиг Blackbox Exporter
- prometheus.service, alertmanager.service, blackbox_exporter.service — юниты

## Как проверить:
- Открыть http://10.0.2.15:8428/vmui/ — это UI VictoriaMetrics
- Ввести запрос probe_success{site="prod"} — должно вернуть 1 
- Открыть http://10.0.2.15:9090/targets — все таргеты UP
- Открыть http://10.0.2.15:9093 — Alertmanager


## Алертинг (Alertmanager)

Alertmanager отправляет email-уведомления через SMTP Яндекса.
Маршрутизация зависит от severity алерта.

### Каналы оповещения

- critical → shamanchukov+critical@yandex.ru
- warning  → shamanchukov+warning@yandex.ru

### Маршрутизация

В alertmanager.yml два дочерних маршрута:

- severity = "critical" → receiver email-critical
- severity = "warning"  → receiver email-warning

### Правила алертов (alerts.yml)

- CMSSiteDown (critical): probe_success{job="blackbox"} == 0 дольше 30 секунд
- CMSSlowResponse (warning): probe_duration_seconds{job="blackbox"} > 0.5 дольше 1 минуты

### SMTP

- Сервер: smtp.yandex.ru:465
- TLS: включён (порт 465 — SMTPS)
- Аутентификация: пароль приложения
  (в репозитории замаскирован как REPLACE_WITH_APP_PASSWORD)

### Как проверить

- amtool alert add test_critical severity=critical alertname=TestCritical
- amtool alert add test_warning severity=warning alertname=TestWarning
- Письма приходят на разные адреса: +critical и +warning

