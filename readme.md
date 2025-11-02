# Production руководство по установке и настройке SeaweedFS кластера

## Содержание

1. [Введение и архитектура SeaweedFS](#введение-и-архитектура-seaweedfs)
2. [Требования к окружению](#требования-к-окружению)
3. [Планирование кластера](#планирование-кластера)
4. [Установка и настройка Master серверов](#установка-и-настройка-master-серверов)
5. [Установка и настройка Volume серверов](#установка-и-настройка-volume-серверов)
6. [Настройка Filer](#настройка-filer)
7. [Настройка репликации и Rack Awareness](#настройка-репликации-и-rack-awareness)
8. [Балансировка нагрузки](#балансировка-нагрузки)
9. [Мониторинг и метрики](#мониторинг-и-метрики)
10. [Backup и восстановление](#backup-и-восстановление)
11. [Оптимизация производительности](#оптимизация-производительности)
12. [Troubleshooting](#troubleshooting)
13. [Production рекомендации](#production-рекомендации)

---

## Введение и архитектура SeaweedFS

SeaweedFS — это высокопроизводительная распределенная файловая система, ориентированная на хранение больших объемов данных. Архитектура "collection-first" делает ее идеальной для multi-tenant окружений.

### Ключевые компоненты

- **Master Server** — управляет mapping'ом файлов к volume серверам
- **Volume Server** — хранит данные и обслуживает файловые операции
- **Filer** — предоставляет POSIX-like файловый интерфейс
- **S3 Gateway** — S3-совместимый API
- **WebDAV Server** — WebDAV протокол для монтирования

### Преимущества для production

- **Простая архитектура** — легче в управлении чем Ceph/HDFS
- **Высокая производительность** — минимальные накладные расходы
- **Эффективное использование места** — дедупликация и компрессия
- **Гибкая репликация** — настраиваемая на уровне коллекций
- **S3 совместимость** — легкая интеграция с существующими приложениями

---

## Требования к окружению

### Аппаратные требования

**Master серверы:**
- CPU: 2-4 ядра
- RAM: 4-8 GB
- Диск: 50-100 GB SSD (для метаданных)
- Сеть: 1 Gbit/s

**Volume серверы:**
- CPU: 8-16 ядер
- RAM: 16-64 GB (зависит от количества томов)
- Диск: 4-24 HDD/SSD, минимум 1 TB каждый
- Сеть: 10 Gbit/s (рекомендуется)

**Filer серверы:**
- CPU: 4-8 ядер
- RAM: 8-32 GB
- Диск: 100-500 GB SSD
- Сеть: 1-10 Gbit/s

### Программные требования

- ОС: Linux (Ubuntu 20.04+, CentOS 7+, Debian 10+)
- Файловая система: XFS (рекомендуется) или EXT4
- Java: OpenJDK 8+ (для мониторинга через Prometheus)
- Firewall: открытые порты 9333, 8080, 8888

### Сетевые порты

- **Master**: 9333 (gRPC), 19333 (HTTP)
- **Volume**: 8080 (HTTP), 18080 (gRPC)
- **Filer**: 8888 (HTTP), 18888 (gRPC)

---

## Планирование кластера

### Пример production топологии
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Master Node     │    │ Master Node     │    │ Master Node     │
│ (Leader)        │◄──►│ (Follower)      │◄──►│ (Follower)      │
│ master-1:9333   │    │ master-2:9333   │    │ master-3:9333   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
          │                     │                      │
          └─────────────────────┼──────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────┴─────────┐ ┌─────────┴─────────┐ ┌─────────┴─────────┐
│ Volume Rack 1     │ │ Volume Rack 2     │ │ Volume Rack 3     │
│ volume-1{1..3}    │ │ volume-2{1..3}    │ │ volume-3{1..3}    │
└───────────────────┘ └───────────────────┘ └───────────────────┘
          │                     │                     │
┌─────────┴─────────┐ ┌─────────┴─────────┐ ┌─────────┴─────────┐
│ Filer-1           │ │ Filer-2           │ │ Filer-3           │
│ filer-1:8888      │ │ filer-2:8888      │ │ filer-3:8888      │
└───────────────────┘ └───────────────────┘ └───────────────────┘

```

### Расчет capacity

**Формула:** `Общий объем = (Количество volume серверов × Дисков на сервер × Размер диска) / Репликация`

**Пример для 9 volume серверов:**
- 9 серверов × 8 дисков × 4 TB = 288 TB raw
- При репликации 3x: 288 TB / 3 = 96 TB usable
- При репликации 2x: 288 TB / 2 = 144 TB usable

---

## Установка и настройка Master серверов

### Установка бинарного файла

```bash
# На всех master нодах
cd /usr/local/bin
wget https://github.com/seaweedfs/seaweedfs/releases/latest/download/linux_amd64.tar.gz
tar -xzf linux_amd64.tar.gz
chmod +x weed
Конфигурация systemd для Master
Создайте файл /etc/systemd/system/seaweedfs-master.service:

ini
[Unit]
Description=SeaweedFS Master Server
After=network.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs
ExecStart=/usr/local/bin/weed master \
  -ip=master-1 \
  -port=9333 \
  -mdir=/data/seaweedfs/metadata \
  -peers=master-1:9333,master-2:9333,master-3:9333 \
  -defaultReplication=001 \
  -volumeSizeLimitMB=30000 \
  -metricsAddress=0.0.0.0:9091 \
  -volumePreallocate

Restart=always
RestartSec=30
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Конфигурационные файлы
Создайте /etc/seaweedfs/master.conf:

yaml
# Master server configuration
ip: "master-1"
port: 9333
metricsPort: 9091

# Cluster peers
peers: 
  - "master-1:9333"
  - "master-2:9333" 
  - "master-3:9333"

# Storage policies
defaultReplication: "001"
volumeSizeLimitMB: 30000
garbageThreshold: 0.3
pulseSeconds: 10

# Security
whiteList:
  - "10.0.0.0/8"
  - "192.168.0.0/16"
Запуск Master кластера
bash
# Создание пользователя и директорий
sudo useradd -r -s /bin/false seaweedfs
sudo mkdir -p /data/seaweedfs/{metadata,volumes}
sudo chown -R seaweedfs:seaweedfs /data/seaweedfs

# Запуск на всех master нодах
sudo systemctl daemon-reload
sudo systemctl enable seaweedfs-master
sudo systemctl start seaweedfs-master

# Проверка статуса
sudo systemctl status seaweedfs-master
weed shell -master=master-1:9333 "cluster.ps"
Проверка кластера Master
bash
# Проверка здоровья кластера
curl http://master-1:9333/cluster/status?pretty=y

# Проверка leader election
curl http://master-1:9333/cluster/raft?pretty=y

# Статус через weed shell
weed shell -master=master-1:9333 "volume.list"
Установка и настройка Volume серверов
Конфигурация systemd для Volume Server
Создайте /etc/systemd/system/seaweedfs-volume.service:

ini
[Unit]
Description=SeaweedFS Volume Server
After=network.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs
ExecStart=/usr/local/bin/weed volume \
  -ip=volume-1-1 \
  -port=8080 \
  -mserver=master-1:9333,master-2:9333,master-3:9333 \
  -dir=/data/seaweedfs/volumes \
  -max=100 \
  -rack=rack-1 \
  -dataCenter=dc1 \
  -compactionMBps=100 \
  -metricsAddress=0.0.0.0:9092 \
  -index=leveldb

Restart=always
RestartSec=30
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Многодисковая конфигурация
Для использования нескольких дисков:

bash
# Создание директорий для каждого диска
sudo mkdir -p /data/{disk1,disk2,disk3,disk4}/seaweedfs/volumes
sudo chown -R seaweedfs:seaweedfs /data/*/seaweedfs

# Запуск с несколькими директориями
ExecStart=/usr/local/bin/weed volume \
  -ip=volume-1-1 \
  -port=8080 \
  -mserver=master-1:9333,master-2:9333,master-3:9333 \
  -dir=/data/disk1/seaweedfs/volumes,/data/disk2/seaweedfs/volumes,/data/disk3/seaweedfs/volumes,/data/disk4/seaweedfs/volumes \
  -max=25 \
  -rack=rack-1 \
  -dataCenter=dc1
Настройка rack awareness
bash
# Volume серверы в разных rack
# Rack 1
-volume -ip=volume-1-1 -rack=rack-1 -dataCenter=dc1 ...
-volume -ip=volume-1-2 -rack=rack-1 -dataCenter=dc1 ...

# Rack 2  
-volume -ip=volume-2-1 -rack=rack-2 -dataCenter=dc1 ...
-volume -ip=volume-2-2 -rack=rack-2 -dataCenter=dc1 ...

# Rack 3
-volume -ip=volume-3-1 -rack=rack-3 -dataCenter=dc1 ...
-volume -ip=volume-3-2 -rack=rack-3 -dataCenter=dc1 ...
Запуск Volume серверов
bash
# На всех volume нодах
sudo systemctl daemon-reload
sudo systemctl enable seaweedfs-volume
sudo systemctl start seaweedfs-volume

# Проверка
sudo systemctl status seaweedfs-volume
curl http://volume-1-1:8080/status?pretty=y
Проверка Volume распределения
bash
# Проверка томов в кластере
weed shell -master=master-1:9333 "volume.list"

# Проверка распределения по rack
curl http://master-1:9333/dir/status?pretty=y
Настройка Filer
Конфигурация systemd для Filer
Создайте /etc/systemd/system/seaweedfs-filer.service:

ini
[Unit]
Description=SeaweedFS Filer Server
After=network.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs
ExecStart=/usr/local/bin/weed filer \
  -ip=filer-1 \
  -port=8888 \
  -master=master-1:9333,master-2:9333,master-3:9333 \
  -dataCenter=dc1 \
  -defaultReplicaPlacement=001 \
  -metricsAddress=0.0.0.0:9093 \
  -leveldb2.databaseDir=/data/seaweedfs/filer \
  -s3 \
  -s3.port=8333

Restart=always
RestartSec=30
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Настройка S3 Gateway
bash
# Запуск filer с S3 поддержкой
ExecStart=/usr/local/bin/weed filer \
  -ip=filer-1 \
  -port=8888 \
  -master=master-1:9333,master-2:9333,master-3:9333 \
  -s3 \
  -s3.port=8333 \
  -s3.domainName=s3.example.com \
  -s3.allowEmptyFolder=true \
  -s3.allowDeleteBucketNotEmpty=true
Настройка базы данных метаданных
LevelDB (рекомендуется для production):

bash
-leveldb2.databaseDir=/data/seaweedfs/filer
-leveldb2.compression=snappy
MySQL/PostgreSQL для распределенного filer:

bash
-filer.db=mysql
-filer.mysql.host=mysql-host:3306
-filer.mysql.database=seaweedfs
-filer.mysql.username=seaweedfs
-filer.mysql.password=your-password
Запуск Filer
bash
# Создание директорий
sudo mkdir -p /data/seaweedfs/filer
sudo chown -R seaweedfs:seaweedfs /data/seaweedfs/filer

# Запуск
sudo systemctl daemon-reload
sudo systemctl enable seaweedfs-filer
sudo systemctl start seaweedfs-filer

# Проверка
curl http://filer-1:8888/?pretty=y
Настройка репликации и Rack Awareness
Схемы репликации
SeaweedFS поддерживает гибкие схемы репликации:

000 — нет репликации

001 — репликация в том же rack

010 — репликация в том же DC, разные rack

100 — репликация в разные DC

200 — две реплики в разных DC

Настройка replication policy
bash
# Глобальная настройка при запуске master
-defaultReplication=010

# Или на уровне коллекции
weed shell -master=master-1:9333 "volume.create -replication=010 -collection=important-data"
Rack и Data Center конфигурация
bash
# Volume серверы с разными rack/DC
-volume -ip=10.0.1.11 -rack=rack1 -dataCenter=dc1
-volume -ip=10.0.1.12 -rack=rack1 -dataCenter=dc1
-volume -ip=10.0.2.11 -rack=rack2 -dataCenter=dc1  
-volume -ip=10.0.3.11 -rack=rack3 -dataCenter=dc1
-volume -ip=10.1.1.11 -rack=rack1 -dataCenter=dc2
Проверка распределения
bash
# Проверка topology
curl http://master-1:9333/cluster/status?pretty=y

# Проверка распределения томов
weed shell -master=master-1:9333 "volume.check.disk"
Балансировка нагрузки
Nginx для Master серверов
nginx
upstream seaweedfs_masters {
    least_conn;
    server master-1:9333 max_fails=3 fail_timeout=30s;
    server master-2:9333 max_fails=3 fail_timeout=30s;
    server master-3:9333 max_fails=3 fail_timeout=30s;
}

server {
    listen 9333;
    location / {
        proxy_pass http://seaweedfs_masters;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
Nginx для Filer серверов
nginx
upstream seaweedfs_filers {
    least_conn;
    server filer-1:8888 max_fails=3 fail_timeout=30s;
    server filer-2:8888 max_fails=3 fail_timeout=30s;
    server filer-3:8888 max_fails=3 fail_timeout=30s;
}

server {
    listen 8888;
    client_max_body_size 100G;
    
    location / {
        proxy_pass http://seaweedfs_filers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }
}
S3 Gateway балансировка
nginx
upstream seaweedfs_s3 {
    least_conn;
    server filer-1:8333 max_fails=3 fail_timeout=30s;
    server filer-2:8333 max_fails=3 fail_timeout=30s;
    server filer-3:8333 max_fails=3 fail_timeout=30s;
}

server {
    listen 8333;
    server_name s3.example.com;
    
    location / {
        proxy_pass http://seaweedfs_s3;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
Мониторинг и метрики
Prometheus конфигурация
yaml
# prometheus.yml
scrape_configs:
  - job_name: 'seaweedfs_master'
    static_configs:
      - targets: 
        - 'master-1:9091'
        - 'master-2:9091' 
        - 'master-3:9091'
    metrics_path: /metrics
    scrape_interval: 30s

  - job_name: 'seaweedfs_volume'
    static_configs:
      - targets:
        - 'volume-1-1:9092'
        - 'volume-1-2:9092'
        - 'volume-2-1:9092'
    scrape_interval: 30s

  - job_name: 'seaweedfs_filer'
    static_configs:
      - targets:
        - 'filer-1:9093'
        - 'filer-2:9093'
    scrape_interval: 30s
Ключевые метрики для мониторинга
Master метрики:

seaweedfs_master_leader — является ли нода лидером

seaweedfs_master_volume_count — количество томов

seaweedfs_master_ec_shard_count — EC shards

seaweedfs_master_max_volume_id — максимальный ID тома

Volume метрики:

seaweedfs_volume_disk_size — размер диска

seaweedfs_volume_disk_used — использованное место

seaweedfs_volume_write_request_count — запросы на запись

seaweedfs_volume_read_request_count — запросы на чтение

Filer метрики:

seaweedfs_filer_request_count — количество запросов

seaweedfs_filer_error_count — ошибки

Grafana дашборды
Используйте официальные дашборды SeaweedFS или создайте свои на основе ключевых метрик.

Health checks
bash
#!/bin/bash
# health-check.sh

# Проверка master
curl -f http://master-1:9333/cluster/status > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "Master unhealthy"
    exit 1
fi

# Проверка filer
curl -f http://filer-1:8888/ > /dev/null 2>&1  
if [ $? -ne 0 ]; then
    echo "Filer unhealthy"
    exit 1
fi

echo "All services healthy"
exit 0
Backup и восстановление
Filer backup
bash
# Полный backup filer метаданных
weed backup -server=filer-1:8888 -dir=/backup/seaweedfs/$(date +%Y%m%d)

# Инкрементальный backup
weed backup -server=filer-1:8888 -dir=/backup/seaweedfs/incremental -since=$(date -d "1 day ago" +%s)
Volume data backup
bash
# Резервное копирование томов через rsync
rsync -av /data/seaweedfs/volumes/ backup-server:/backup/seaweedfs/volumes/

# Или использование LVM snapshots
lvcreate -L 10G -s -n seaweedfs-snapshot /dev/vg0/seaweedfs-volumes
Восстановление из backup
bash
# Восстановление filer метаданных
weed restore -server=filer-1:8888 -dir=/backup/seaweedfs/20240101

# Восстановление volume данных
rsync -av backup-server:/backup/seaweedfs/volumes/ /data/seaweedfs/volumes/
Replication как backup стратегия
bash
# Настройка cross-DC репликации
-volume -ip=volume-dc1-1 -dataCenter=dc1
-volume -ip=volume-dc2-1 -dataCenter=dc2

# Создание томов с cross-DC репликацией
weed shell -master=master-1:9333 "volume.create -replication=100"
Оптимизация производительности
Оптимизация Volume серверов
bash
# Параметры запуска volume сервера
-volume \
  -compactionMBps=200 \          # Ограничение скорости компрессии
  -idleTimeout=30 \              # Таймаут idle томов
  -fileSizeLimitMB=1024 \        # Лимит размера файла
  -index=leveldb \               # Более быстрый индекс
  -read.redirect=true \          # Редирект чтения
  -cpuProfile= \                 # Отключение CPU профилирования в prod
  -memProfile=                   # Отключение memory профилирования
Оптимизация Filer
bash
# Параметры filer для высокой нагрузки
-filer \
  -concurrentWalSize=1000 \      # Размер WAL
  -leveldb2.blockCacheSize=256 \ # Кэш для LevelDB
  -leveldb2.writeBufferSize=64 \ # Write buffer
  -disableDirListing=false \     # Включить листинг директорий
  -maxMB=0                       # Без ограничения размера файла
Системные оптимизации
bash
# Увеличение лимитов ядра
echo 'fs.file-max = 1000000' >> /etc/sysctl.conf
echo 'net.core.somaxconn = 65536' >> /etc/sysctl.conf
echo 'vm.swappiness = 1' >> /etc/sysctl.conf

# Оптимизация XFS
mkfs.xfs -f -i size=512 /dev/sdb1
mount -o noatime,nodiratime,logbufs=8,logbsize=256k /dev/sdb1 /data/volumes
Оптимизация сети
bash
# Настройка TCP для 10G сетей
echo 'net.core.rmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 87380 67108864' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 67108864' >> /etc/sysctl.conf
Troubleshooting
Распространенные проблемы и решения
Проблема: Master не может выбрать лидера

bash
# Проверка RAFT статуса
curl http://master-1:9333/cluster/raft?pretty=y

# Решение: перезапуск master серверов
sudo systemctl restart seaweedfs-master
Проблема: Volume серверы не регистрируются

bash
# Проверка подключения к master
telnet master-1 9333

# Проверка логов volume сервера
sudo journalctl -u seaweedfs-volume -f

# Решение: проверка firewall и правильности IP адресов
Проблема: Медленная запись

bash
# Проверка дискового IO
iostat -x 1

# Проверка сети
iperf3 -c volume-server-1

# Решение: оптимизация компрессии и проверка дисков
Проблема: Filer не отвечает

bash
# Проверка базы данных filer
weed shell -filer=filer-1:8888 "fs.du /"

# Решение: проверка подключения к master и объема памяти
Полезные команды диагностики
bash
# Статус кластера
weed shell -master=master-1:9333 "cluster.ps"

# Статус томов
weed shell -master=master-1:9333 "volume.list"

# Проверка дисков
weed shell -master=master-1:9333 "volume.check.disk"

# Статус filer
weed shell -filer=filer-1:8888 "fs.du /"

# Статистика использования
curl http://master-1:9333/cluster/status?pretty=y
Логирование и отладка
bash
# Включение debug логирования
-master.defaultLogLevel=debug
-volume.defaultLogLevel=info
-filer.defaultLogLevel=warn

# Просмотр логов в реальном времени
sudo journalctl -u seaweedfs-master -f
sudo journalctl -u seaweedfs-volume -f  
sudo journalctl -u seaweedfs-filer -f
Production рекомендации
Архитектурные рекомендации
Всегда используйте нечетное количество master серверов (3, 5, 7)

Размещайте volume серверы в разных rack/DC для отказоустойчивости

Используйте отдельные сети для данных и управления

Настройте мониторинг всех компонентов с алертами

Регулярно тестируйте процедуры восстановления

Безопасность
bash
# Ограничение доступа к master
-whiteList=10.0.0.0/8,192.168.0.0/16

# Использование TLS
-tls.enabled=true
-tls.certFile=/path/to/cert.pem
-tls.keyFile=/path/to/key.pem

# Аутентификация для S3
-s3.config=/etc/seaweedfs/s3-credentials.json
Резервное копирование
Регулярный backup filer метаданных

Snapshot томов для point-in-time восстановления

Cross-DC репликация для критичных данных

Тестирование восстановления каждые 3 месяца

Мониторинг емкости
bash
#!/bin/bash
# capacity-alert.sh

CAPACITY=$(curl -s http://master-1:9333/cluster/status | jq '.Topology.used / .Topology.max * 100')
if (( $(echo "$CAPACITY > 85" | bc -l) )); then
    echo "WARNING: Cluster capacity at ${CAPACITY}%"
    # Отправка алерта
fi
Обновление версий
Тестируйте обновления в staging среде

Обновляйте volume серверы по одному

Обновляйте master серверы после volume

Обновляйте filer серверы в последнюю очередь

Имейте rollback план

Процедуры обслуживания
Добавление нового volume сервера:

bash
# Просто запустите новый volume сервер
sudo systemctl start seaweedfs-volume
Удаление volume сервера:

bash
# Дождитесь миграции данных
weed shell -master=master-1:9333 "volume.balance -force"

# Остановите сервер
sudo systemctl stop seaweedfs-volume
Замена диска:

bash
# Остановите volume сервер
sudo systemctl stop seaweedfs-volume

# Замените диск и перемонтируйте
sudo umount /data/disk1
# ... замена диска ...
sudo mount /data/disk1

# Запустите volume сервер
sudo systemctl start seaweedfs-volume
Заключение
SeaweedFS предоставляет мощную и гибкую платформу для распределенного хранения данных в production средах. Ключевые преимущества:

Простота эксплуатации — минимальные накладные расходы на управление

Высокая производительность — оптимизированная для больших файлов

Гибкая репликация — настраиваемая на уровне коллекций

S3 совместимость — легкая интеграция с существующими приложениями

Следуя рекомендациям этого руководства, вы сможете развернуть отказоустойчивый, производительный и легко масштабируемый кластер SeaweedFS для любых production workload'ов.

Удачи в эксплуатации вашего SeaweedFS кластера!
