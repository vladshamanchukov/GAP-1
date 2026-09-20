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
