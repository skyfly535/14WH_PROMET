# Настройка мониторинга.

Настроить дашборд с 4-мя графиками

1. память;
2. процессор;
3. диск;
4. сеть.

## Настройка системы мониторинга prometheus - grafana.

Для начала на WM c CentOS 7 выполняем скрипт по развертыванию связки Node_exporter + Prometheus + Grafana

```
      cd
      # создаем рабочую деррикторию для установки
      mkdir prometheus
      cd prometheus/
      # выкачиваем и распаковываем установочный пакет prometheus
      wget https://github.com/prometheus/prometheus/releases/download/v2.39.1/prometheus-2.39.1.linux-amd64.tar.gz
      tar -xvf prometheus-2.39.1.linux-amd64.tar.gz
      # выкачиваем и распаковываем установочный пакет node_exporter
      wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz
      tar -xvf node_exporter-1.4.0.linux-amd64.tar.gz
      rm node_exporter-1.4.0.linux-amd64.tar.gz #можно не удалаять
      # выкачиваем и распаковываем установочный пакет grafana
      wget https://dl.grafana.com/oss/release/grafana-9.2.1-1.x86_64.rpm
      # создаем тех пользователя для запуска Prometheus
      useradd --no-create-home --shell /usr/sbin/nologin RKPrometheus
      # создаем тех пользователя для запуска Node_exporter
      useradd --no-create-home --shell /bin/false RKNode_exporter
      # создаем рабочие папки в системных каталогах 
      mkdir -v {/etc,/var/lib}/prometheus
      # даем права на созданные папки тез пользователю
      chown -v RKPrometheus: {/etc,/var/lib}/prometheus
      # переходим в скаченный каталог node_exporter
      cd /root/prometheus/node_exporter-1.4.0.linux-amd64/
      # копируем необходимые файлы из скаченног пакаета node_exporter в системные дерректории
      rsync -P node_exporter /usr/local/bin #(из каталога где дистриб Node_exporter)
       # даем права на эти дерриктории тех юзеру
      chown -v RKNode_exporter: /usr/local/bin/node_exporter
      cd /root/prometheus/prometheus-2.39.1.linux-amd64/
      # копируем необходимые файлы из скаченног пакаета prometheus в системные дерректории
      rsync -P prometheus /usr/local/bin #(из каталога где дистриб Node_exporter)#
      rsync -P promtool /usr/local/bin
      # даем права на эти дерриктории тех юзеру
      chown -v RKPrometheus: /usr/local/bin/prometheus
      chown -v RKPrometheus: /usr/local/bin/promtool
      # копируем приложенный Unit systemd для запуска prometheus
      \cp -u ./Project/prometheus.service /etc/systemd/system/
      # копируем приложенный Unit systemd для запуска node_exporter
      \cp -u ./Project/node_exporter.service /etc/systemd/system/
      # копируем приложенный файл настроек prometheus
      \cp -u ./Project/prometheus.yml /etc/prometheus/
      cd /root/prometheus/prometheus-2.39.1.linux-amd64/
      cp -r consoles/ /etc/prometheus/
      cp -r console_libraries/ /etc/prometheus/
      cd /etc/prometheus/
      chown -v  RKPrometheus: prometheus.yml
      # даем права на файл настроек тех юзеру на файл настроек 
      chown -v  RKPrometheus: prometheus.yml
      # даем права на рабочие каталоги тех 
      chown -v  RKPrometheus: consoles
      chown -v  RKPrometheus: console_libraries/
      chmod -R 777 /var/lib/prometheus
      # запускаем node_exporter.service
      systemctl enable --now node_exporter.service
      # запускаем prometheus.service
      systemctl enable --now prometheus.service
      cd /root/prometheus/
      rm prometheus-2.39.1.linux-amd64.tar.gz
      # устанавоиваем grafana
      yum install ./grafana-9.2.1-1.x86_64.rpm
      # запускаем grafana
      systemctl enable --now grafana-server.service
```
Необходимо проверить, чтоб `firewalld` и `selinux` не блокировали рабочие порты!!!

Теперь идем на Grafana `http://192.168.152.162:3000` и производим первичный вход.

Дальше добавляем источника данных:

1. кликаем по иконке `Configuration - Data Sources`
2. добавляем источник, нажав по `Add data source`
3. находим и выбираем Prometheus, кликаем по `Select`
4. задаем параметры для подключения к Prometheus (имя, IP сервера, порт)
5. сохраняем настройки `Save & Test`.

## Создаем свой Dashboard.

Нажимаем Dashboard - New dashboard.

Нажимаем Add a new panel.

Далее необходимо заполнить следующие поля:

- Title - название панели
- Description - описние панели
- PromQL query - текст запроса для выборки данных из Prometheus.

### Память

Процент занятой оперативной памяти. PromQL query:

```
100 * ((avg_over_time(node_memory_MemFree_bytes[5m]) + avg_over_time(node_memory_Cached_bytes[5m]) + avg_over_time(node_memory_Buffers_bytes[5m])) / avg_over_time(node_memory_MemTotal_bytes[5m]))
```

### Процессор

Процент утилизации процессорного времени. PromQL query:

```
100 - (avg by (instance)(irate(node_cpu_seconds_total{job="node_exporter",mode="idle"}[5m])) * 100)
```

### Диск 

Объем свободного места на накопителе в процентах. PromQL query:

```
node_filesystem_free_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"}
```

### Сеть

Входящий трафик за последние 5 минут. PromQL query:

```
rate(node_network_receive_bytes_total[5m]) * 8 / 1024 / 1024
```
Не забываем все сохранить.

Получаем результат:

![Dash_Prometheus](https://user-images.githubusercontent.com/114483769/227204297-8f7eaaf8-c66f-4738-b77e-d61e656e5a6c.jpeg)
