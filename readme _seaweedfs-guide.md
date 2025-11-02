#### Уровень 1: Репликация (встроенная)

Не является полноценным backup, но обеспечивает:
- Защиту от сбоя дисков/серверов
- Мгновенное восстановление
- Не защищает от логических ошибок (случайное удаление)

#### Уровень 2: Snapshot volumes

```bash
# Создание snapshot конкретного volume
curl "http://volume1:8080/admin/volume/snapshot?volumeId=5&destination=/backup/volume5"

# Автоматический snapshot через cron
#!/bin/bash
# /usr/local/bin/volume-snapshot.sh

BACKUP_DIR="/backup/volumes/$(date +%Y%m%d)"
MASTER="master1:9333"

mkdir -p ${BACKUP_DIR}

# Получить список всех volumes
VOLUMES=$(curl -s "http://${MASTER}/vol/status" | jq -r '.Volumes.DataCenters[].Racks[].DataNodes[].VolumeIds' | tr ',' '\n')

for VOL in ${VOLUMES}; do
    # Найти volume server для этого volume
    SERVER=$(curl -s "http://${MASTER}/vol/lookup?volumeId=${VOL}" | jq -r '.locations[0].url')
    
    # Создать snapshot
    curl "http://${SERVER}/admin/volume/snapshot?volumeId=${VOL}&destination=${BACKUP_DIR}"
done
```

#### Уровень 3: Export через Filer

```bash
# Экспорт всех файлов через Filer API
#!/bin/bash
# /usr/local/bin/filer-backup.sh

FILER="filer1:8888"
BACKUP_ROOT="/backup/filer/$(date +%Y%m%d)"
SOURCE_PATH="/"

# Рекурсивное скачивание
wget --mirror --no-parent --no-host-directories \
     --cut-dirs=0 --directory-prefix=${BACKUP_ROOT} \
     "http://${FILER}${SOURCE_PATH}"

# Или с помощью rclone
rclone sync seaweedfs:/ ${BACKUP_ROOT} \
  --config /etc/rclone/rclone.conf \
  --transfers 32 \
  --checkers 64 \
  --log-file /var/log/rclone-backup.log
```

#### Уровень 4: S3 репликация

```bash
# Настройка репликации в другой S3-совместимый storage
# rclone конфигурация

[seaweedfs]
type = s3
provider = Other
endpoint = http://s3-gw1:8333
access_key_id = admin_access_key
secret_access_key = admin_secret_key

[aws-s3-backup]
type = s3
provider = AWS
region = us-east-1
access_key_id = AWS_ACCESS_KEY
secret_access_key = AWS_SECRET_KEY

# Синхронизация
rclone sync seaweedfs:my-bucket aws-s3-backup:backup-bucket \
  --transfers 16 \
  --s3-upload-concurrency 8
```

### Backup метаданных

#### Master метаданные

```bash
# Backup master данных
#!/bin/bash
BACKUP_DIR="/backup/master/$(date +%Y%m%d)"
MASTER_DIR="/var/lib/seaweedfs/master"

mkdir -p ${BACKUP_DIR}

# Остановить master (или сделать snapshot на лету)
systemctl stop seaweedfs-master

# Копировать данные
rsync -av ${MASTER_DIR}/ ${BACKUP_DIR}/

systemctl start seaweedfs-master

# Сжать архив
tar -czf ${BACKUP_DIR}.tar.gz ${BACKUP_DIR}
```

#### Filer метаданные (PostgreSQL)

```bash
# Backup PostgreSQL базы Filer
#!/bin/bash
BACKUP_DIR="/backup/filer-db"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_DIR}

# Dump базы данных
pg_dump -h localhost -U seaweedfs -d seaweedfs_filer \
  -F c -f ${BACKUP_DIR}/filer_${TIMESTAMP}.dump

# Ротация старых backup'ов (хранить 30 дней)
find ${BACKUP_DIR} -name "filer_*.dump" -mtime +30 -delete

# Загрузить в удаленное хранилище
rclone copy ${BACKUP_DIR}/filer_${TIMESTAMP}.dump remote-s3:backups/filer/
```

### Автоматизация backup

#### Комплексный backup скрипт

```bash
#!/bin/bash
# /usr/local/bin/seaweedfs-full-backup.sh

set -e

BACKUP_ROOT="/backup/seaweedfs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_ROOT}/${TIMESTAMP}"
MASTER="master1:9333"
FILER="filer1:8888"
RETENTION_DAYS=30

mkdir -p ${BACKUP_DIR}/{volumes,master,filer-db,filer-data}

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a ${BACKUP_DIR}/backup.log
}

log "Starting SeaweedFS backup"

# 1. Backup Master метаданных
log "Backing up Master metadata"
systemctl stop seaweedfs-master
rsync -av /var/lib/seaweedfs/master/ ${BACKUP_DIR}/master/
systemctl start seaweedfs-master

# 2. Backup Filer метаданных
log "Backing up Filer database"
pg_dump -h localhost -U seaweedfs -d seaweedfs_filer \
  -F c -f ${BACKUP_DIR}/filer-db/filer.dump

# 3. Backup volumes (snapshot)
log "Creating volume snapshots"
VOLUMES=$(curl -s "http://${MASTER}/vol/status" | \
  jq -r '.Volumes.DataCenters[].Racks[].DataNodes[].VolumeIds' | \
  tr ',' '\n' | sort -u)

for VOL in ${VOLUMES}; do
    SERVER=$(curl -s "http://${MASTER}/vol/lookup?volumeId=${VOL}" | \
      jq -r '.locations[0].url')
    
    log "Snapshotting volume ${VOL} from ${SERVER}"
    curl -s "http://${SERVER}/admin/volume/snapshot?volumeId=${VOL}&destination=${BACKUP_DIR}/volumes/"
done

# 4. Backup конфигураций
log "Backing up configurations"
mkdir -p ${BACKUP_DIR}/config
cp -r /etc/seaweedfs/* ${BACKUP_DIR}/config/
cp /etc/systemd/system/seaweedfs-* ${BACKUP_DIR}/config/

# 5. Создать манифест
log "Creating backup manifest"
cat > ${BACKUP_DIR}/manifest.json <<EOF
{
  "timestamp": "${TIMESTAMP}",
  "backup_type": "full",
  "components": {
    "master": "$(du -sh ${BACKUP_DIR}/master | cut -f1)",
    "filer_db": "$(du -sh ${BACKUP_DIR}/filer-db | cut -f1)",
    "volumes": "$(du -sh ${BACKUP_DIR}/volumes | cut -f1)",
    "config": "$(du -sh ${BACKUP_DIR}/config | cut -f1)"
  },
  "volumes_count": $(echo "${VOLUMES}" | wc -l),
  "total_size": "$(du -sh ${BACKUP_DIR} | cut -f1)"
}
EOF

# 6. Сжать backup
log "Compressing backup"
tar -czf ${BACKUP_ROOT}/seaweedfs_${TIMESTAMP}.tar.gz -C ${BACKUP_ROOT} ${TIMESTAMP}

# 7. Загрузить в удаленное хранилище
log "Uploading to remote storage"
rclone copy ${BACKUP_ROOT}/seaweedfs_${TIMESTAMP}.tar.gz remote-s3:backups/seaweedfs/

# 8. Очистка старых backup'ов
log "Cleaning old backups"
find ${BACKUP_ROOT} -name "seaweedfs_*.tar.gz" -mtime +${RETENTION_DAYS} -delete
find ${BACKUP_ROOT} -maxdepth 1 -type d -mtime +${RETENTION_DAYS} -exec rm -rf {} \;

# 9. Проверка целостности
log "Verifying backup integrity"
tar -tzf ${BACKUP_ROOT}/seaweedfs_${TIMESTAMP}.tar.gz > /dev/null

log "Backup completed successfully"
echo "${TIMESTAMP}" > ${BACKUP_ROOT}/LATEST
```

#### Cron расписание

```bash
# Ежедневный полный backup в 2:00 AM
0 2 * * * /usr/local/bin/seaweedfs-full-backup.sh

# Ежечасный incremental backup (только изменения)
0 * * * * /usr/local/bin/seaweedfs-incremental-backup.sh
```

### Восстановление из backup

#### Восстановление Master

```bash
#!/bin/bash
# /usr/local/bin/restore-master.sh

BACKUP_FILE="/backup/seaweedfs/seaweedfs_20240115_020000.tar.gz"
EXTRACT_DIR="/tmp/seaweedfs-restore"

# Остановить Master
systemctl stop seaweedfs-master

# Извлечь backup
mkdir -p ${EXTRACT_DIR}
tar -xzf ${BACKUP_FILE} -C ${EXTRACT_DIR}

# Найти директорию с backup'ом
BACKUP_DIR=$(find ${EXTRACT_DIR} -name "master" -type d | head -1 | xargs dirname)

# Восстановить метаданные Master
rm -rf /var/lib/seaweedfs/master/*
rsync -av ${BACKUP_DIR}/master/ /var/lib/seaweedfs/master/

# Восстановить конфигурацию
cp ${BACKUP_DIR}/config/seaweedfs-master.service /etc/systemd/system/

# Перезагрузить и запустить
systemctl daemon-reload
systemctl start seaweedfs-master

# Проверить статус
systemctl status seaweedfs-master
```

#### Восстановление Volume

```bash
#!/bin/bash
# /usr/local/bin/restore-volumes.sh

BACKUP_DIR="/backup/seaweedfs/20240115_020000/volumes"
VOLUME_DIR="/mnt/seaweedfs/volume1"

# Остановить Volume server
systemctl stop seaweedfs-volume

# Очистить старые данные (ОСТОРОЖНО!)
read -p "This will delete all existing volumes. Continue? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    echo "Aborted"
    exit 1
fi

rm -rf ${VOLUME_DIR}/*

# Восстановить volumes
rsync -av ${BACKUP_DIR}/ ${VOLUME_DIR}/

# Установить права
chown -R weed:weed ${VOLUME_DIR}

# Запустить сервер
systemctl start seaweedfs-volume

# Проверить volumes
curl "http://localhost:8080/status"
```

#### Восстановление Filer

```bash
#!/bin/bash
# /usr/local/bin/restore-filer.sh

BACKUP_FILE="/backup/seaweedfs/20240115_020000/filer-db/filer.dump"

# Остановить Filer
systemctl stop seaweedfs-filer

# Удалить старую базу
sudo -u postgres psql <<EOF
DROP DATABASE IF EXISTS seaweedfs_filer;
CREATE DATABASE seaweedfs_filer;
GRANT ALL PRIVILEGES ON DATABASE seaweedfs_filer TO seaweedfs;
EOF

# Восстановить dump
pg_restore -h localhost -U seaweedfs -d seaweedfs_filer ${BACKUP_FILE}

# Запустить Filer
systemctl start seaweedfs-filer

# Проверить
curl "http://localhost:8888/"
```

### Disaster Recovery план

#### Сценарий 1: Полная потеря одного data center

```bash
# 1. Проверить доступность резервного DC
curl "http://master-dc2:9333/cluster/status"

# 2. Перенаправить клиентов на резервный DC
# Обновить DNS или load balancer

# 3. Проверить репликацию данных
weed shell -master="master-dc2:9333"
> volume.fix.replication -n

# 4. Если нужно, восстановить under-replicated volumes
> volume.fix.replication
```

#### Сценарий 2: Полная потеря кластера

```bash
# 1. Подготовить новые серверы
# 2. Установить SeaweedFS на всех узлах
# 3. Восстановить Master из последнего backup
/usr/local/bin/restore-master.sh

# 4. Запустить Master кластер
# На всех master серверах
systemctl start seaweedfs-master

# 5. Восстановить Volumes на всех volume серверах
/usr/local/bin/restore-volumes.sh

# 6. Восстановить Filer
/usr/local/bin/restore-filer.sh

# 7. Проверить целостность
weed shell
> volume.check.disk
> volume.fix.replication
```

### Тестирование восстановления

```bash
#!/bin/bash
# /usr/local/bin/test-restore.sh

# Ежемесячный тест восстановления
TEST_ENV="/tmp/seaweedfs-test"
BACKUP_FILE=$(cat /backup/seaweedfs/LATEST)

log() {
    echo "[$(date)] $1" | tee -a /var/log/restore-test.log
}

log "Starting restore test with backup: ${BACKUP_FILE}"

# 1. Создать тестовое окружение
mkdir -p ${TEST_ENV}/{master,volume,filer}

# 2. Восстановить в тестовом окружении
# ... (команды восстановления)

# 3. Запустить компоненты в тестовом режиме
weed master -mdir=${TEST_ENV}/master -port=19333 &
MASTER_PID=$!

sleep 5

weed volume -dir=${TEST_ENV}/volume -mserver=localhost:19333 -port=18080 &
VOLUME_PID=$!

# 4. Проверить доступность
sleep 10
curl "http://localhost:19333/cluster/status" || { log "FAILED: Master not responding"; exit 1; }
curl "http://localhost:18080/status" || { log "FAILED: Volume not responding"; exit 1; }

# 5. Тест записи/чтения
TEST_FILE="/tmp/test-file-$.txt"
echo "Test data" > ${TEST_FILE}

FILE_ID=$(curl -s "http://localhost:19333/dir/assign" | jq -r '.fid')
curl -F "file=@${TEST_FILE}" "http://localhost:18080/${FILE_ID}"
curl "http://localhost:18080/${FILE_ID}" | grep "Test data" || { log "FAILED: Data integrity check"; exit 1; }

# 6. Очистка
kill ${MASTER_PID} ${VOLUME_PID}
rm -rf ${TEST_ENV}

log "Restore test PASSED"
```

---

## Оптимизация производительности

### Оптимизация на уровне ОС

#### Kernel параметры

```bash
# /etc/sysctl.conf

# Увеличение лимитов файлов
fs.file-max = 2097152
fs.nr_open = 2097152

# TCP оптимизации
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 16384

# Оптимизация для SSD
vm.swappiness = 10
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5

# Применить
sysctl -p
```

#### IO Scheduler

```bash
# Для SSD/NVMe используйте noop или none
echo "none" > /sys/block/nvme0n1/queue/scheduler

# Для HDD используйте deadline
echo "deadline" > /sys/block/sda/queue/scheduler

# Постоянная настройка через udev
cat > /etc/udev/rules.d/60-scheduler.rules <<EOF
# SSD/NVMe
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"

# HDD
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="deadline"
EOF
```

#### Файловая система

```bash
# XFS монтирование с оптимизациями
mount -o noatime,nodiratime,nobarrier,logbufs=8,logbsize=256k /dev/sdb /mnt/seaweedfs/volume1

# В /etc/fstab
/dev/sdb /mnt/seaweedfs/volume1 xfs noatime,nodiratime,nobarrier,logbufs=8,logbsize=256k 0 0

# Для ext4
mount -o noatime,nodiratime,data=writeback,barrier=0,nobh /dev/sdb /mnt/seaweedfs/volume1
```

### Оптимизация Master

#### Параметры запуска

```bash
ExecStart=/usr/local/bin/weed master \
  -port=9333 \
  -mdir=/var/lib/seaweedfs/master \
  -peers=master1:9333,master2:9333,master3:9333 \
  -defaultReplication=010 \
  -volumeSizeLimitMB=30000 \
  -volumePreallocate=true \
  -garbageThreshold=0.2 \
  -pulseSeconds=3 \
  -metricsPort=9324
```

**Ключевые параметры производительности:**

- `volumePreallocate`: Предварительное выделение места для volumes (снижает fragmentation)
- `garbageThreshold`: Порог для garbage collection (0.2 = 20% удаленных файлов)
- `pulseSeconds`: Частота heartbeat (меньше = быстрее обнаружение сбоев, но больше нагрузка)

#### Memory tuning

```bash
# Для больших кластеров увеличьте heap Go
# В systemd service
Environment="GOGC=100"
Environment="GOMEMLIMIT=8GiB"
```

### Оптимизация Volume

#### Параметры запуска

```bash
ExecStart=/usr/local/bin/weed volume \
  -port=8080 \
  -dir=/mnt/seaweedfs/volume1 \
  -max=200 \
  -mserver=master1:9333,master2:9333,master3:9333 \
  -dataCenter=dc1 \
  -rack=rack1 \
  -compactionMBps=100 \
  -readMode=local \
  -minFreeSpacePercent=5 \
  -metricsPort=9325 \
  -index=memory
```

**Ключевые параметры:**

- `compactionMBps`: Скорость compaction (100 = 100 MB/s, балансируйте с рабочей нагрузкой)
- `readMode=local`: Приоритет локального чтения
- `minFreeSpacePercent`: Минимальный процент свободного места
- `index=memory`: Хранить индекс в памяти (быстрее, но требует больше RAM)

#### Compaction стратегии

```bash
# Агрессивная compaction (больше CPU/IO, меньше места)
-garbageThreshold=0.1 -compactionMBps=200

# Консервативная compaction (меньше нагрузка, больше места)
-garbageThreshold=0.5 -compactionMBps=50

# Ночная compaction через cron
0 2 * * * curl "http://volume1:8080/admin/volume/compact/full"
```

#### Размещение volumes

```bash
# Разделение hot и cold данных
# Hot tier (SSD)
-dir=/mnt/ssd/hot -max=100 -disk=ssd

# Cold tier (HDD)
-dir=/mnt/hdd/cold -max=500 -disk=hdd

# Автоматическое tier'ирование через TTL
# В Master конфигурации
collection.create -name=temp -replication=001 -ttl=7d -diskType=ssd
collection.create -name=archive -replication=010 -diskType=hdd
```

### Оптимизация Filer

#### Database tuning (PostgreSQL)

```sql
-- /etc/postgresql/14/main/postgresql.conf

-- Память
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 64MB
maintenance_work_mem = 1GB

-- Checkpoint
checkpoint_completion_target = 0.9
wal_buffers = 16MB
min_wal_size = 1GB
max_wal_size = 4GB

-- Query planning
random_page_cost = 1.1  -- для SSD
effective_io_concurrency = 200

-- Connections
max_connections = 200

-- Logging (для отладки производительности)
log_min_duration_statement = 1000  -- логировать запросы >1s
```

#### Индексы PostgreSQL

```sql
-- Оптимизация индексов Filer
CREATE INDEX CONCURRENTLY idx_dirhash_name ON filer_mapping(dirhash, name);
CREATE INDEX CONCURRENTLY idx_directory ON filer_mapping(directory);

-- Анализ и обновление статистики
VACUUM ANALYZE filer_mapping;

-- Автовакуум настройки
ALTER TABLE filer_mapping SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_analyze_scale_factor = 0.02
);
```

#### Filer кеширование

```bash
ExecStart=/usr/local/bin/weed filer \
  -port=8888 \
  -master=master1:9333,master2:9333,master3:9333 \
  -collection=filer \
  -dirListLimit=100000 \
  -dirListCacheLimit=100000 \
  -cacheCapacityMB=10000 \
  -cacheDir=/var/cache/seaweedfs
```

### Оптимизация S3 Gateway

```bash
ExecStart=/usr/local/bin/weed s3 \
  -port=8333 \
  -filer=filer1:8888,filer2:8888 \
  -config=/etc/seaweedfs/s3.json \
  -domainName=s3.example.com \
  -cert.file=/etc/ssl/certs/s3.crt \
  -key.file=/etc/ssl/private/s3.key \
  -metricsPort=9327 \
  -cacheCapacityMB=1000
```

### Network оптимизации

#### Jumbo Frames

```bash
# Включить jumbo frames (MTU 9000)
ip link set eth0 mtu 9000

# Постоянная настройка
cat >> /etc/network/interfaces <<EOF
auto eth0
iface eth0 inet static
    mtu 9000
EOF
```

#### TCP BBR

```bash
# Включить BBR congestion control
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p

# Проверить
sysctl net.ipv4.tcp_congestion_control
```

### Benchmarking

#### Volume Server benchmark

```bash
# Write benchmark
weed benchmark -server=volume1:8080 -n=100000 -size=1024

# Read benchmark
weed benchmark -server=volume1:8080 -read -n=100000

# Mixed workload
weed benchmark -server=volume1:8080 -n=100000 -size=1024 -readPercent=70
```

#### Filer benchmark

```bash
# fio benchmark через mounted filer
fio --name=seaweedfs-test \
    --directory=/mnt/seaweedfs \
    --size=10G \
    --numjobs=4 \
    --bs=1M \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randrw \
    --rwmixread=70 \
    --runtime=300 \
    --group_reporting
```

#### S3 benchmark (s3-benchmark)

```bash
# Установка
go install github.com/dvassallo/s3-benchmark@latest

# Тест
s3-benchmark \
  -endpoint=http://s3-gw1:8333 \
  -access-key=admin_access_key \
  -secret-key=admin_secret_key \
  -bucket=test-bucket \
  -size=1M \
  -requests=10000 \
  -concurrent=32
```

### Мониторинг производительности

```promql
# Latency по percentiles
histogram_quantile(0.50, rate(weed_volume_request_duration_seconds_bucket[5m]))  # p50
histogram_quantile(0.95, rate(weed_volume_request_duration_seconds_bucket[5m]))  # p95
histogram_quantile(0.99, rate(weed_volume_request_duration_seconds_bucket[5m]))  # p99

# IOPS
rate(weed_volume_request_total[1m])

# Throughput
rate(weed_volume_read_bytes_total[1m]) + rate(weed_volume_write_bytes_total[1m])

# Error rate
rate(weed_volume_request_errors_total[1m]) / rate(weed_volume_request_total[1m])
```

---

## Troubleshooting

### Общие проблемы и решения

#### Проблема 1: "No free volumes"

**Симптомы:**
```
Error: No free volumes
```

**Причина:** Все volumes заполнены или достигнут лимит volumes

**Решение:**

```bash
# 1. Проверить свободное место
curl "http://master1:9333/vol/status" | jq '.Volumes.Free'

# 2. Увеличить лимит volumes на серверах
# В systemd service
-max=200  # было 100

# 3. Добавить новые volume серверы

# 4. Запустить compaction для освобождения места
curl "http://volume1:8080/admin/volume/compact/full"
```

#### Проблема 2: High latency

**Диагностика:**

```bash
# Проверить метрики latency
curl "http://volume1:9325/metrics" | grep duration_seconds

# Проверить IO wait
iostat -x 1

# Проверить network latency
ping -c 100 volume2
```

**Решения:**

```bash
# 1. Оптимизировать IO scheduler
echo "none" > /sys/block/nvme0n1/queue/scheduler

# 2. Проверить disk health
smartctl -a /dev/sda

# 3. Уменьшить compaction rate
-compactionMBps=50

# 4. Включить read-only режим для проблемного volume
curl "http://volume1:8080/admin/volume/readonly?volumeId=5&readonly=true"
```

#### Проблема 3: Split brain в Master кластере

**Симптомы:**
```
Multiple leaders detected
Inconsistent cluster state
```

**Решение:**

```bash
# 1. Остановить все master серверы
for host in master1 master2 master3; do
    ssh ${host} "systemctl stop seaweedfs-master"
done

# 2. Очистить Raft данные на всех кроме одного
# ОСТОРОЖНО! Делайте backup
for host in master2 master3; do
    ssh ${host} "rm -rf /var/lib/seaweedfs/master/raft"
done

# 3. Запустить по очереди
ssh master1 "systemctl start seaweedfs-master"
sleep 10
ssh master2 "systemctl start seaweedfs-master"
sleep 10
ssh master3 "systemctl start seaweedfs-master"

# 4. Проверить статус
curl "http://master1:9333/cluster/status"
```

#### Проблема 4: Volume corruption

**Симптомы:**
```
Error reading volume file
Checksum mismatch
```

**Диагностика:**

```bash
# Проверить целостность volume
weed shell
> volume.check.disk -volumeId=5

# Проверить filesystem
umount /mnt/seaweedfs/volume1
xfs_repair /dev/sdb
mount /mnt/seaweedfs/volume1
```

**Решение:**

```bash
# 1. Пометить volume как readonly
curl "http://volume1:8080/admin/volume/readonly?volumeId=5&readonly=true"

# 2. Скопировать с реплики
weed shell
> volume.copy -volumeId=5 -source=volume2:8080 -target=volume1:8080

# 3. Или восстановить из backup
/usr/local/bin/restore-volumes.sh
```

#### Проблема 5: Out of memory

**Симптомы:**
```
killed by OOM killer
insufficient memory
```

**Диагностика:**

```bash
# Проверить использование памяти
free -h
ps aux | grep# Production-ready SeaweedFS: Полное руководство по установке и настройке

## Содержание

1. [Введение](#введение)
2. [Архитектура SeaweedFS](#архитектура-seaweedfs)
3. [Почему SeaweedFS](#почему-seaweedfs)
4. [Требования к окружению](#требования-к-окружению)
5. [Планирование кластера](#планирование-кластера)
6. [Установка Master серверов](#установка-master-серверов)
7. [Установка Volume серверов](#установка-volume-серверов)
8. [Настройка Filer и S3 Gateway](#настройка-filer-и-s3-gateway)
9. [Репликация и отказоустойчивость](#репликация-и-отказоустойчивость)
10. [Балансировка нагрузки](#балансировка-нагрузки)
11. [Мониторинг и алертинг](#мониторинг-и-алертинг)
12. [Backup и восстановление](#backup-и-восстановление)
13. [Оптимизация производительности](#оптимизация-производительности)
14. [Troubleshooting](#troubleshooting)
15. [Production чеклист](#production-чеклист)

---

## Введение

SeaweedFS — это высокопроизводительная распределенная файловая система с S3-совместимым API, специально разработанная для хранения миллионов файлов различного размера. В отличие от традиционных решений вроде Ceph или MinIO, SeaweedFS использует архитектуру "конструктора", где метаданные отделены от данных, что позволяет масштабироваться без деградации производительности.

### Ключевые преимущества

- **Отделение метаданных от данных** — метаданные хранятся в базе данных, данные — в оптимизированных "суперблоках"
- **Нет ограничений файловых систем** — миллионы файлов хранятся в нескольких больших файлах
- **Гибкая репликация** — настраивается на уровне коллекций с rack/DC awareness
- **Простое масштабирование** — добавление дисков любого размера без ребалансировки всего кластера
- **S3-совместимость** — полная поддержка S3 API через встроенный gateway
- **Низкие требования к ресурсам** — эффективное использование CPU, RAM и сети

### Типичные сценарии использования

- Хранение сотен миллионов мелких файлов (изображения, документы, логи)
- Multi-tenant окружения с разными требованиями к хранению
- Замена дорогих облачных S3 хранилищ собственной инфраструктурой
- Построение data lake с гарантированной сохранностью данных
- Высоконагруженные системы с требованиями к низкой latency

---

## Архитектура SeaweedFS

### Компоненты системы

SeaweedFS состоит из четырех основных компонентов, каждый из которых выполняет специфическую роль:

#### 1. Master Server

Центральный координатор кластера, отвечающий за:

- Управление топологией кластера и метаданными volume
- Выделение file IDs для новых файлов
- Отслеживание доступных volume серверов и их свободного места
- Координация репликации данных
- Предоставление информации о расположении файлов

**Критически важно**: Master должен быть высокодоступным. Рекомендуется развертывание минимум 3 master серверов с использованием Raft consensus для обеспечения отказоустойчивости.

#### 2. Volume Server

Сервер хранения данных, который:

- Хранит фактические данные файлов в volumes (больших файлах-контейнерах)
- Обрабатывает запросы на чтение и запись
- Управляет локальной репликацией
- Поддерживает множественные диски и директории
- Выполняет автоматическое сжатие (compaction) данных

Каждый volume server может управлять несколькими volumes, и каждый volume имеет фиксированный размер (обычно 30GB).

#### 3. Filer

Опциональный компонент, предоставляющий:

- Иерархическую файловую систему (POSIX-подобный интерфейс)
- Поддержку директорий и path-based доступа
- Метаданные файлов (владелец, права, timestamps)
- WebDAV и REST API для доступа к файлам
- Поддержку различных backend'ов для хранения метаданных (LevelDB, MySQL, PostgreSQL, Redis)

#### 4. S3 Gateway

Обеспечивает полную совместимость с Amazon S3 API:

- Bucket операции (создание, удаление, листинг)
- Object операции (PUT, GET, DELETE, HEAD)
- Multipart uploads для больших файлов
- S3 authentication и ACLs
- Integration с существующими S3-совместимыми инструментами

### Архитектурная схема

```
┌─────────────────────────────────────────────────────────┐
│                    Client Applications                   │
│              (S3 API, REST API, WebDAV)                 │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐       ┌───────▼────────┐
│  S3 Gateway    │       │     Filer      │
│  (Port 8333)   │       │  (Port 8888)   │
└───────┬────────┘       └───────┬────────┘
        │                        │
        └────────────┬───────────┘
                     │
        ┌────────────▼────────────┐
        │    Master Cluster       │
        │  (Raft Consensus)       │
        │  Master1, Master2, M3   │
        │     (Port 9333)         │
        └────────────┬────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐       ┌───────▼────────┐
│ Volume Server1 │       │ Volume Server2 │
│  (Port 8080)   │◄─────►│  (Port 8080)   │
└────────────────┘       └────────────────┘
        │                         │
┌───────▼────────┐       ┌───────▼────────┐
│  Disk Storage  │       │  Disk Storage  │
│   (Volumes)    │       │   (Volumes)    │
└────────────────┘       └────────────────┘
```

### Поток данных

**Запись файла:**

1. Клиент запрашивает у Master сервера file ID и адрес доступного Volume
2. Master возвращает file ID и список Volume серверов (включая реплики)
3. Клиент напрямую загружает данные на указанные Volume сервера
4. Volume сервера реплицируют данные между собой согласно политике репликации

**Чтение файла:**

1. Клиент запрашивает у Master адрес Volume сервера по file ID
2. Master возвращает список доступных Volume серверов с данными
3. Клиент напрямую читает данные с ближайшего Volume сервера

**Ключевое преимущество**: Master участвует только в координации, данные передаются напрямую между клиентом и Volume серверами, что исключает bottleneck и обеспечивает высокую пропускную способность.

---

## Почему SeaweedFS

### Сравнение с альтернативами

| Характеристика | SeaweedFS | Ceph | MinIO | GlusterFS |
|----------------|-----------|------|-------|-----------|
| Архитектура | Master + Volume | CRUSH + OSD | Erasure Coding | Distributed Hash |
| Метаданные | Централизованные | Распределенные | Централизованные | Распределенные |
| Малые файлы | Отлично | Плохо | Хорошо | Удовлетворительно |
| S3 API | Нативный | Через RGW | Нативный | Через плагин |
| Сложность настройки | Низкая | Высокая | Низкая | Средняя |
| Требования к ресурсам | Низкие | Высокие | Средние | Средние |
| Масштабируемость | Линейная | Хорошая | Хорошая | Ограниченная |

### Когда выбирать SeaweedFS

**Идеально подходит для:**

- Проектов с миллионами мелких файлов (фото, документы, логи)
- Ситуаций, где важна простота развертывания и поддержки
- Когда нужна высокая производительность при ограниченных ресурсах
- Замены дорогих облачных S3 хранилищ
- Систем с требованиями к low latency

**Не рекомендуется для:**

- Хранения очень больших файлов (>1TB) — лучше использовать Ceph
- Когда критична консистентность данных уровня distributed transactions
- Если требуется POSIX-совместимость без Filer компонента
- Блочного хранилища для виртуальных машин

### Производственные кейсы

**Хранение изображений (CDN)**

Компания с 500M+ изображений мигрировала с AWS S3, получив:
- Снижение стоимости на 80%
- Улучшение latency на 40%
- Полный контроль над данными

**Система логирования**

Обработка 100TB+ логов в день с:
- Записью 50K запросов/сек
- Средним размером файла 100KB
- Retention 90 дней

**Архив документов**

Долгосрочное хранение 200M+ документов:
- Полная S3 совместимость
- Автоматическая репликация
- Простое масштабирование

---

## Требования к окружению

### Минимальные системные требования

#### Master Server

- **CPU**: 2 ядра
- **RAM**: 4GB (для метаданных ~10M файлов)
- **Disk**: 50GB SSD для метаданных
- **Network**: 1Gbps
- **OS**: Linux (Ubuntu 20.04+, CentOS 7+, Debian 10+)

#### Volume Server

- **CPU**: 4+ ядра (зависит от нагрузки)
- **RAM**: 8GB+ (2GB на каждые 100 volumes)
- **Disk**: Зависит от объема данных, рекомендуется SSD/NVMe для hot data
- **Network**: 10Gbps для production
- **OS**: Linux с поддержкой XFS или ext4

#### Filer Server

- **CPU**: 2-4 ядра
- **RAM**: 8GB+
- **Disk**: 100GB+ SSD для метаданных
- **Network**: 1-10Gbps
- **OS**: Linux

### Рекомендуемая конфигурация для production

#### Small deployment (до 10TB, 10M файлов)

- 3 Master сервера: 2 CPU, 4GB RAM, 50GB SSD
- 3 Volume сервера: 8 CPU, 16GB RAM, 4TB HDD each
- 2 Filer сервера: 4 CPU, 8GB RAM, 100GB SSD
- 1 S3 Gateway: 4 CPU, 8GB RAM

#### Medium deployment (до 100TB, 100M файлов)

- 3 Master сервера: 4 CPU, 8GB RAM, 100GB SSD
- 6-10 Volume серверов: 16 CPU, 32GB RAM, 10TB HDD each
- 3 Filer сервера: 8 CPU, 16GB RAM, 200GB SSD
- 2-3 S3 Gateway: 8 CPU, 16GB RAM

#### Large deployment (500TB+, 1B+ файлов)

- 5 Master серверов: 8 CPU, 16GB RAM, 500GB SSD
- 20+ Volume серверов: 32 CPU, 64GB RAM, 20TB SSD/HDD mix
- 5+ Filer серверов: 16 CPU, 32GB RAM, 500GB SSD
- 5+ S3 Gateway: 16 CPU, 32GB RAM
- Dedicated load balancers (HAProxy/Nginx)

### Дисковые подсистемы

**Для Master серверов:**
- Обязательно SSD/NVMe
- RAID1 или RAID10 для надежности
- Низкая latency критична для метаданных

**Для Volume серверов:**
- Hot data: NVMe SSD (высокая IOPS)
- Warm data: SATA SSD (баланс цена/производительность)
- Cold data: HDD RAID6/RAID10 (емкость)
- No RAID0 — используйте репликацию SeaweedFS

**Для Filer:**
- SSD для базы метаданных
- Быстрый random I/O важен

### Сетевые требования

- Минимум 1Gbps между компонентами
- Рекомендуется 10Gbps для production
- Низкая latency между Volume серверами (<1ms)
- Separate VLANs для storage traffic
- Jumbo frames (MTU 9000) для увеличения throughput

### Операционная система

**Рекомендуемые дистрибутивы:**

- Ubuntu 20.04 LTS / 22.04 LTS
- Debian 11 / 12
- CentOS 8 Stream / Rocky Linux 8+
- RHEL 8+

**Необходимые настройки ядра:**

```bash
# Увеличение лимитов файлов
echo "fs.file-max = 1000000" >> /etc/sysctl.conf

# Оптимизация сети
echo "net.core.rmem_max = 134217728" >> /etc/sysctl.conf
echo "net.core.wmem_max = 134217728" >> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 87380 67108864" >> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 65536 67108864" >> /etc/sysctl.conf

# Применить изменения
sysctl -p
```

**Ulimits:**

```bash
# /etc/security/limits.conf
* soft nofile 1000000
* hard nofile 1000000
* soft nproc 65535
* hard nproc 65535
```

### Зависимости

SeaweedFS написан на Go и распространяется как статический бинарный файл без внешних зависимостей. Однако для production окружения потребуются:

- **Опционально**: PostgreSQL/MySQL для Filer метаданных
- **Опционально**: Redis для кеширования
- **Рекомендуется**: Prometheus для мониторинга
- **Рекомендуется**: Grafana для визуализации метрик

---

## Планирование кластера

### Определение размера кластера

#### Расчет количества Volume серверов

```
Volume Servers = (Total Data Size × Replication Factor) / Disk Size per Server
```

Пример: 100TB данных, репликация 3x, 10TB диск на сервер:
```
(100TB × 3) / 10TB = 30 volume серверов
```

#### Расчет количества Master серверов

- **Минимум**: 3 для Raft consensus
- **Рекомендуется**: 3-5 для большинства deployments
- **Large scale**: 5-7 для >1B файлов

#### Расчет памяти для Master

Приблизительно 1GB RAM на каждые 1M файлов для метаданных. Для 100M файлов потребуется ~100GB RAM, распределенных между master серверами.

### Топология кластера

#### Rack awareness

SeaweedFS поддерживает rack-aware репликацию для защиты от сбоев на уровне стойки:

```
DataCenter1/
├── Rack1/
│   ├── Volume1
│   └── Volume2
└── Rack2/
    ├── Volume3
    └── Volume4
```

#### Data Center awareness

Для географически распределенных deployments:

```
/
├── DC-East/
│   ├── Rack1/
│   └── Rack2/
└── DC-West/
    ├── Rack1/
    └── Rack2/
```

### Схемы репликации

SeaweedFS использует формат репликации: `XYZ`

- **X**: Количество копий в других data centers
- **Y**: Количество копий в других racks в том же DC
- **Z**: Количество копий на других серверах в том же rack

**Примеры:**

- `000` — без репликации (только для тестирования)
- `001` — 2 копии на разных серверах в одном rack (минимум для production)
- `010` — 2 копии в разных racks
- `100` — 2 копии в разных data centers
- `200` — 3 копии в разных data centers
- `110` — 1 копия в другом DC + 1 в другом rack
- `111` — максимальная защита: копии в другом DC, rack и server

### Стратегии размещения данных

#### Scenario 1: Single Data Center

```
DC1
├── Rack1: 3 Volume Servers
├── Rack2: 3 Volume Servers
└── Rack3: 3 Volume Servers

Репликация: 010 (разные racks)
```

#### Scenario 2: Multi Data Center

```
DC1 (Primary)
├── Rack1: 4 VS
└── Rack2: 4 VS

DC2 (Secondary)
├── Rack1: 4 VS
└── Rack2: 4 VS

Репликация: 100 (копия в другом DC)
```

#### Scenario 3: Hybrid (Hot + Cold)

```
Hot Tier (SSD):
- Replication: 001
- 30 дней retention
- High IOPS

Warm Tier (HDD):
- Replication: 010
- 90 дней retention
- Medium IOPS

Cold Tier (Archive):
- Replication: 100
- Unlimited retention
- Low cost
```

### Collections (Коллекции)

Collections позволяют группировать данные с разными требованиями:

```bash
# Создание коллекций
weed shell <<EOF
collection.create -name=images -replication=001
collection.create -name=videos -replication=010
collection.create -name=backups -replication=100 -ttl=90d
EOF
```

**Параметры коллекций:**

- **replication**: Схема репликации
- **ttl**: Time To Live (автоудаление старых данных)
- **diskType**: hdd/ssd для размещения на определенных дисках
- **dataCenter**: Привязка к конкретному DC

### Capacity planning

#### Расчет IOPS

```
Total IOPS = Disk IOPS × Number of Disks / Replication Factor
```

Пример: 10 дисков по 1000 IOPS, репликация 3x:
```
(10 × 1000) / 3 ≈ 3333 IOPS
```

#### Расчет пропускной способности

```
Throughput = Network Bandwidth × Utilization Factor
```

Для 10Gbps сети с утилизацией 70%:
```
10Gbps × 0.7 = 7Gbps ≈ 875 MB/s
```

#### Планирование роста

Учитывайте:
- Рост данных: +30-50% ежегодно (типично)
- Запас по IOPS: +50% для burst нагрузки
- Запас по емкости: +30% для fragmentation и overhead

### Пример конфигурации production кластера

#### Medium-scale deployment

**Архитектура:**

- 3 Master серверов (HA)
- 9 Volume серверов (3 per rack)
- 2 Filer серверов (HA)
- 2 S3 Gateway (load balanced)

**Размещение:**

```
Rack1:
- Master1: 4 CPU, 8GB RAM, 100GB SSD
- Volume1: 16 CPU, 32GB RAM, 10TB SSD
- Volume2: 16 CPU, 32GB RAM, 10TB SSD
- Volume3: 16 CPU, 32GB RAM, 10TB SSD

Rack2:
- Master2: 4 CPU, 8GB RAM, 100GB SSD
- Volume4: 16 CPU, 32GB RAM, 10TB SSD
- Volume5: 16 CPU, 32GB RAM, 10TB SSD
- Volume6: 16 CPU, 32GB RAM, 10TB SSD

Rack3:
- Master3: 4 CPU, 8GB RAM, 100GB SSD
- Volume7: 16 CPU, 32GB RAM, 10TB SSD
- Volume8: 16 CPU, 32GB RAM, 10TB SSD
- Volume9: 16 CPU, 32GB RAM, 10TB SSD

External (Edge):
- Filer1: 8 CPU, 16GB RAM, 200GB SSD
- Filer2: 8 CPU, 16GB RAM, 200GB SSD
- S3-GW1: 8 CPU, 16GB RAM
- S3-GW2: 8 CPU, 16GB RAM
```

**Параметры:**

- Емкость: 90TB raw (30TB с репликацией 010)
- IOPS: ~90K (предполагая SSD 10K IOPS)
- Throughput: ~8GB/s read, ~4GB/s write
- Файлов: до 100M
- Репликация: 010 (rack-aware)

---

## Установка Master серверов

### Загрузка бинарного файла

```bash
# Скачивание последней версии
WEED_VERSION="3.70"
wget https://github.com/seaweedfs/seaweedfs/releases/download/${WEED_VERSION}/linux_amd64.tar.gz

# Распаковка
tar -xzf linux_amd64.tar.gz

# Установка
sudo mv weed /usr/local/bin/
sudo chmod +x /usr/local/bin/weed

# Проверка версии
weed version
```

### Подготовка окружения

```bash
# Создание пользователя
sudo useradd -r -s /bin/false weed

# Создание директорий
sudo mkdir -p /var/lib/seaweedfs/master
sudo mkdir -p /var/log/seaweedfs
sudo mkdir -p /etc/seaweedfs

# Настройка прав
sudo chown -R weed:weed /var/lib/seaweedfs
sudo chown -R weed:weed /var/log/seaweedfs
sudo chown -R weed:weed /etc/seaweedfs
```

### Конфигурация systemd для Master

Создайте файл `/etc/systemd/system/seaweedfs-master.service`:

```ini
[Unit]
Description=SeaweedFS Master Server
After=network.target
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=weed
Group=weed
ExecStart=/usr/local/bin/weed master \
  -port=9333 \
  -mdir=/var/lib/seaweedfs/master \
  -peers=master1:9333,master2:9333,master3:9333 \
  -defaultReplication=010 \
  -volumeSizeLimitMB=30000 \
  -metricsPort=9324

Restart=on-failure
RestartSec=5
LimitNOFILE=1000000

StandardOutput=journal
StandardError=journal
SyslogIdentifier=seaweedfs-master

[Install]
WantedBy=multi-user.target
```

### Параметры запуска Master

**Основные параметры:**

- `-port=9333`: HTTP API порт
- `-mdir`: Директория для хранения метаданных
- `-peers`: Список всех master серверов для Raft cluster
- `-defaultReplication`: Схема репликации по умолчанию

**Дополнительные параметры:**

- `-volumeSizeLimitMB`: Максимальный размер volume (default 30000)
- `-volumePreallocate`: Предварительное выделение места
- `-garbageThreshold`: Порог для запуска garbage collection (default 0.3)
- `-pulseSeconds`: Интервал heartbeat от volume серверов (default 5)
- `-metricsPort`: Порт для Prometheus метрик

### Настройка Raft кластера

#### На первом Master (master1)

```bash
# Запуск первого master
sudo systemctl start seaweedfs-master
sudo systemctl enable seaweedfs-master

# Проверка статуса
sudo systemctl status seaweedfs-master

# Проверка логов
sudo journalctl -u seaweedfs-master -f
```

#### На втором Master (master2)

Отредактируйте `/etc/systemd/system/seaweedfs-master.service`, добавив `-ip`:

```ini
ExecStart=/usr/local/bin/weed master \
  -ip=master2 \
  -port=9333 \
  -mdir=/var/lib/seaweedfs/master \
  -peers=master1:9333,master2:9333,master3:9333 \
  -defaultReplication=010 \
  -volumeSizeLimitMB=30000 \
  -metricsPort=9324
```

Запустите сервис:

```bash
sudo systemctl daemon-reload
sudo systemctl start seaweedfs-master
sudo systemctl enable seaweedfs-master
```

#### На третьем Master (master3)

Аналогично master2, но с `-ip=master3`.

### Проверка состояния кластера

```bash
# Проверка leader
curl "http://master1:9333/cluster/status"

# Пример ответа
{
  "IsLeader": true,
  "Leader": "master1:9333",
  "Peers": [
    "master1:9333",
    "master2:9333",
    "master3:9333"
  ]
}
```

```bash
# Проверка всех master серверов
curl "http://master1:9333/dir/status"

# Проверка топологии
curl "http://master1:9333/vol/status"
```

### Безопасность Master серверов

#### JWT Authentication

Создайте security.toml:

```toml
# /etc/seaweedfs/security.toml

[jwt.signing]
key = "your-secret-key-here-min-32-chars"

[jwt.signing.read]
expires_after_seconds = 3600

[jwt.signing.write]
expires_after_seconds = 3600
```

Обновите systemd service:

```ini
ExecStart=/usr/local/bin/weed master \
  -port=9333 \
  -mdir=/var/lib/seaweedfs/master \
  -peers=master1:9333,master2:9333,master3:9333 \
  -defaultReplication=010 \
  -volumeSizeLimitMB=30000 \
  -metricsPort=9324 \
  -jwt=true \
  -jwt.signing.key="your-secret-key-here"
```

#### Firewall настройки

```bash
# Разрешить доступ только с volume серверов
sudo ufw allow from 10.0.1.0/24 to any port 9333 proto tcp
sudo ufw allow from 10.0.1.0/24 to any port 9324 proto tcp
```

### Мониторинг Master серверов

#### Healthcheck endpoint

```bash
# Проверка здоровья
curl "http://master1:9333/cluster/healthz"

# Ожидаемый ответ
{
  "IsLeader": true
}
```

#### Prometheus метрики

Master экспортирует метрики на порту 9324:

```bash
curl "http://master1:9324/metrics"
```

Основные метрики:

- `weed_master_volumes_count`: Количество volumes
- `weed_master_volume_servers_count`: Количество volume серверов
- `weed_master_file_count`: Общее количество файлов
- `weed_master_bytes_total`: Общий объем данных

### Troubleshooting Master

#### Проблема: Master не становится leader

```bash
# Проверить логи
sudo journalctl -u seaweedfs-master -n 100

# Проверить сетевую доступность между master серверами
nc -zv master2 9333
nc -zv master3 9333

# Проверить Raft состояние
curl "http://master1:9333/cluster/status"
```

**Решение**: Убедитесь, что все master сервера могут подключаться друг к другу, и параметр `-peers` идентичен на всех серверах.

#### Проблема: Split brain

```bash
# Симптомы: несколько leaders одновременно
# Решение: остановить все masters и запустить заново

sudo systemctl stop seaweedfs-master  # на всех серверах

# Очистить Raft данные (ОСТОРОЖНО!)
sudo rm -rf /var/lib/seaweedfs/master/*

# Запустить по очереди, начиная с master1
sudo systemctl start seaweedfs-master
```

#### Проблема: Высокая память

```bash
# Проверить количество volumes и файлов
curl "http://master1:9333/vol/status" | jq

# Если метаданных слишком много, увеличьте RAM или:
# - Включите volume pruning
# - Уменьшите volumeSizeLimitMB для создания меньшего количества volumes
```

---

## Установка Volume серверов

### Подготовка дисков

```bash
# Форматирование диска в XFS (рекомендуется)
sudo mkfs.xfs -f -L seaweedfs-data /dev/sdb

# Или ext4
sudo mkfs.ext4 -F -L seaweedfs-data /dev/sdb

# Создание точки монтирования
sudo mkdir -p /mnt/seaweedfs/volume1

# Монтирование
sudo mount /dev/sdb /mnt/seaweedfs/volume1

# Добавление в fstab для автомонтирования
echo "LABEL=seaweedfs-data /mnt/seaweedfs/volume1 xfs defaults,noatime 0 0" | sudo tee -a /etc/fstab

# Настройка прав
sudo chown -R weed:weed /mnt/seaweedfs/volume1
```

### Оптимизация файловой системы

```bash
# XFS оптимизации для больших файлов
sudo mount -o remount,noatime,nodiratime,nobarrier /mnt/seaweedfs/volume1

# Отключение atime обновлений (снижение IO)
# Уже включено через noatime в fstab
```

### Конфигурация systemd для Volume

Создайте файл `/etc/systemd/system/seaweedfs-volume.service`:

```ini
[Unit]
Description=SeaweedFS Volume Server
After=network.target
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=weed
Group=weed
ExecStart=/usr/local/bin/weed volume \
  -port=8080 \
  -dir=/mnt/seaweedfs/volume1 \
  -max=100 \
  -mserver=master1:9333,master2:9333,master3:9333 \
  -dataCenter=dc1 \
  -rack=rack1 \
  -ip=volume1.local \
  -publicUrl=volume1.example.com:8080 \
  -metricsPort=9325 \
  -compactionMBps=50

Restart=on-failure
RestartSec=5
LimitNOFILE=1000000

StandardOutput=journal
StandardError=journal
SyslogIdentifier=seaweedfs-volume

[Install]
WantedBy=multi-user.target
```

### Параметры запуска Volume

**Основные параметры:**

- `-port=8080`: HTTP порт для данных
- `-dir`: Директория для хранения volumes (можно указать несколько через запятую)
- `-max`: Максимальное количество volumes на этой директории
- `-mserver`: Список master серверов

**Топология:**

- `-dataCenter`: Имя data center
- `-rack`: Имя rack
- `-ip`: IP или hostname для внутренней коммуникации
- `-publicUrl`: Публичный URL для доступа клиентов

**Производительность:**

- `-compactionMBps`: Скорость compaction в MB/s (default 50)
- `-fileSizeLimitMB`: Размер volume файла (default 30000)
- `-idleTimeout`: Таймаут для неактивных соединений

### Множественные диски на одном сервере

```bash
# Подготовка нескольких дисков
sudo mkfs.xfs -f /dev/sdb
sudo mkfs.xfs -f /dev/sdc
sudo mkfs.xfs -f /dev/sdd

sudo mkdir -p /mnt/seaweedfs/{volume1,volume2,volume3}

sudo mount /dev/sdb /mnt/seaweedfs/volume1
sudo mount /dev/sdc /mnt/seaweedfs/volume2
sudo mount /dev/sdd /mnt/seaweedfs/volume3

# Обновление systemd service
ExecStart=/usr/local/bin/weed volume \
  -port=8080 \
  -dir=/mnt/seaweedfs/volume1,/mnt/seaweedfs/volume2,/mnt/seaweedfs/volume3 \
  -max=100,100,100 \
  -mserver=master1:9333,master2:9333,master3:9333 \
  -dataCenter=dc1 \
  -rack=rack1 \
  -metricsPort=9325
```

### Disk types (SSD vs HDD)

```bash
# Помечаем диски по типу
ExecStart=/usr/local/bin/weed volume \
  -port=8080 \
  -dir=/mnt/ssd/volume1,/mnt/hdd/volume2 \
  -max=50,100 \
  -disk=ssd,hdd \
  -mserver=master1:9333,master2:9333,master3:9333
```

### Запуск Volume серверов

```bash
# Запуск на каждом volume сервере
sudo systemctl daemon-reload
sudo systemctl start seaweedfs-volume
sudo systemctl enable seaweedfs-volume

# Проверка статуса
sudo systemctl status seaweedfs-volume

# Просмотр логов
sudo journalctl -u seaweedfs-volume -f
```

### Проверка подключения к кластеру

```bash
# На master сервере проверяем топологию
curl "http://master1:9333/vol/status" | jq

# Пример ответа
{
  "Version": "3.70",
  "Volumes": {
    "DataCenters": [
      {
        "Id": "dc1",
        "Racks": [
          {
            "Id": "rack1",
            "DataNodes": [
              {
                "Url": "volume1.local:8080",
                "Volumes": 5,
                "Max": 100,
                "Free": 95,
                "PublicUrl": "volume1.example.com:8080"
              }
            ]
          }
        ]
      }
    ],
    "Free": 95,
    "Max": 100
  }
}
```

### Компакция (Compaction)

Volume сервера автоматически выполняют compaction для удаления помеченных на удаление файлов.

```bash
# Ручной запуск compaction
curl "http://volume1:8080/admin/volume/compact?volumeId=1"

# Настройка автоматической compaction
ExecStart=/usr/local/bin/weed volume \
  ... \
  -compactionMBps=50 \
  -minFreeSpacePercent=1
```

### Балансировка volumes

```bash
# Через weed shell
weed shell

# Балансировка volumes между серверами
> volume.balance -c ALL -force

# Перемещение конкретного volume
> volume.move -volumeId=5 -source=volume1:8080 -target=volume2:8080
```

---

## Настройка Filer и S3 Gateway

### Установка Filer

Filer предоставляет файловую систему с поддержкой путей и директорий поверх SeaweedFS.

#### Выбор метаданных backend

Filer может использовать различные базы данных для метаданных:

- **LevelDB** (default): Локальная embedded база, простая, но не HA
- **PostgreSQL**: Рекомендуется для production, поддержка HA
- **MySQL**: Альтернатива PostgreSQL
- **Redis**: Для высокой производительности, требует persistence
- **Cassandra**: Для очень больших deployments

#### Настройка PostgreSQL для Filer

```bash
# Установка PostgreSQL
sudo apt-get install postgresql postgresql-contrib

# Создание базы и пользователя
sudo -u postgres psql <<EOF
CREATE DATABASE seaweedfs_filer;
CREATE USER seaweedfs WITH ENCRYPTED PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE seaweedfs_filer TO seaweedfs;
EOF
```

#### Конфигурация Filer

Создайте файл `/etc/seaweedfs/filer.toml`:

```toml
[postgres2]
enabled = true
createTable = """
  CREATE TABLE IF NOT EXISTS "%s" (
    dirhash   BIGINT,
    name      VARCHAR(65535),
    directory VARCHAR(65535),
    meta      bytea,
    PRIMARY KEY (dirhash, name)
  );
"""
hostname = "localhost"
port = 5432
username = "seaweedfs"
password = "secure_password"
database = "seaweedfs_filer"
schema = "public"
sslmode = "disable"
connection_max_idle = 100
connection_max_open = 100
connection_max_lifetime_seconds = 0
enableUpsert = true
upsertQuery = """
  INSERT INTO "%[1]s" (dirhash, name, directory, meta) 
  VALUES ($1, $2, $3, $4) 
  ON CONFLICT (dirhash, name) 
  DO UPDATE SET meta = EXCLUDED.meta 
  WHERE "%[1]s".meta != EXCLUDED.meta
"""
```

#### Systemd service для Filer

Создайте файл `/etc/systemd/system/seaweedfs-filer.service`:

```ini
[Unit]
Description=SeaweedFS Filer Server
After=network.target postgresql.service
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=weed
Group=weed
ExecStart=/usr/local/bin/weed filer \
  -port=8888 \
  -master=master1:9333,master2:9333,master3:9333 \
  -dataCenter=dc1 \
  -defaultReplicaPlacement=010 \
  -disableDirListing=false \
  -metricsPort=9326 \
  -collection=filer

Restart=on-failure
RestartSec=5
LimitNOFILE=1000000

StandardOutput=journal
StandardError=journal
SyslogIdentifier=seaweedfs-filer

[Install]
WantedBy=multi-user.target
```

### Параметры запуска Filer

**Основные параметры:**

- `-port=8888`: HTTP API порт
- `-master`: Список master серверов
- `-collection`: Коллекция по умолчанию для файлов
- `-defaultReplicaPlacement`: Схема репликации по умолчанию

**Дополнительные параметры:**

- `-dirListLimit`: Максимум файлов в листинге директории (default 100000)
- `-disableDirListing`: Отключить листинг директорий для безопасности
- `-encryptVolumeData`: Включить шифрование данных
- `-maxMB`: Максимальный размер загружаемого файла

### Запуск Filer

```bash
sudo systemctl daemon-reload
sudo systemctl start seaweedfs-filer
sudo systemctl enable seaweedfs-filer

# Проверка
sudo systemctl status seaweedfs-filer

# Тест доступности
curl "http://filer1:8888/"
```

### Работа с Filer через API

```bash
# Загрузка файла
curl -F "file=@/path/to/file.txt" "http://filer1:8888/path/to/destination/"

# Чтение файла
curl "http://filer1:8888/path/to/destination/file.txt"

# Удаление файла
curl -X DELETE "http://filer1:8888/path/to/destination/file.txt"

# Листинг директории
curl "http://filer1:8888/path/to/directory/?pretty=y"

# Создание директории
curl -X POST "http://filer1:8888/path/to/new/directory/"
```

### Настройка S3 Gateway

S3 Gateway предоставляет полностью совместимый с Amazon S3 API интерфейс.

#### Конфигурация S3

Создайте файл `/etc/seaweedfs/s3.json`:

```json
{
  "identities": [
    {
      "name": "admin",
      "credentials": [
        {
          "accessKey": "admin_access_key",
          "secretKey": "admin_secret_key"
        }
      ],
      "actions": [
        "Admin",
        "Read",
        "Write"
      ]
    },
    {
      "name": "readonly",
      "credentials": [
        {
          "accessKey": "readonly_access_key",
          "secretKey": "readonly_secret_key"
        }
      ],
      "actions": [
        "Read"
      ]
    }
  ]
}
```

#### Systemd service для S3

Создайте файл `/etc/systemd/system/seaweedfs-s3.service`:

```ini
[Unit]
Description=SeaweedFS S3 Gateway
After=network.target seaweedfs-filer.service
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=weed
Group=weed
ExecStart=/usr/local/bin/weed s3 \
  -port=8333 \
  -filer=filer1:8888,filer2:8888 \
  -config=/etc/seaweedfs/s3.json \
  -domainName=s3.example.com \
  -metricsPort=9327

Restart=on-failure
RestartSec=5
LimitNOFILE=1000000

StandardOutput=journal
StandardError=journal
SyslogIdentifier=seaweedfs-s3

[Install]
WantedBy=multi-user.target
```

### Параметры S3 Gateway

**Основные параметры:**

- `-port=8333`: HTTP порт для S3 API
- `-filer`: Список filer серверов (для HA)
- `-config`: Путь к файлу с credentials
- `-domainName`: Доменное имя для bucket subdomain style

**Дополнительные параметры:**

- `-cert.file` / `-key.file`: SSL сертификаты для HTTPS
- `-allowEmptyFolder`: Разрешить пустые папки (для S3 совместимости)
- `-allowDeleteBucketNotEmpty`: Разрешить удаление непустых buckets

### Запуск S3 Gateway

```bash
sudo systemctl daemon-reload
sudo systemctl start seaweedfs-s3
sudo systemctl enable seaweedfs-s3

# Проверка
sudo systemctl status seaweedfs-s3

# Тест доступности
curl "http://s3-gw1:8333/"
```

### Тестирование S3 API

#### С помощью AWS CLI

```bash
# Установка AWS CLI
pip install awscli

# Конфигурация
aws configure
# AWS Access Key ID: admin_access_key
# AWS Secret Access Key: admin_secret_key
# Default region: us-east-1
# Default output format: json

# Или через environment variables
export AWS_ACCESS_KEY_ID=admin_access_key
export AWS_SECRET_ACCESS_KEY=admin_secret_key
export AWS_ENDPOINT_URL=http://s3-gw1:8333

# Создание bucket
aws --endpoint-url=http://s3-gw1:8333 s3 mb s3://test-bucket

# Загрузка файла
aws --endpoint-url=http://s3-gw1:8333 s3 cp /path/to/file.txt s3://test-bucket/

# Листинг объектов
aws --endpoint-url=http://s3-gw1:8333 s3 ls s3://test-bucket/

# Скачивание файла
aws --endpoint-url=http://s3-gw1:8333 s3 cp s3://test-bucket/file.txt /tmp/

# Удаление файла
aws --endpoint-url=http://s3-gw1:8333 s3 rm s3://test-bucket/file.txt

# Удаление bucket
aws --endpoint-url=http://s3-gw1:8333 s3 rb s3://test-bucket
```

#### С помощью s3cmd

```bash
# Установка
sudo apt-get install s3cmd

# Конфигурация ~/.s3cfg
cat > ~/.s3cfg <<EOF
[default]
access_key = admin_access_key
secret_key = admin_secret_key
host_base = s3-gw1:8333
host_bucket = s3-gw1:8333
use_https = False
EOF

# Использование
s3cmd mb s3://test-bucket
s3cmd put file.txt s3://test-bucket/
s3cmd ls s3://test-bucket/
```

### High Availability для Filer и S3

#### Установка нескольких Filer

Запустите идентичные Filer сервера на разных хостах с одинаковой конфигурацией PostgreSQL. Они будут автоматически синхронизироваться через общую базу метаданных.

```bash
# На filer1
sudo systemctl start seaweedfs-filer

# На filer2 (та же конфигурация)
sudo systemctl start seaweedfs-filer
```

#### Load Balancer для Filer

```nginx
# Nginx конфигурация
upstream filer_backend {
    least_conn;
    server filer1:8888 max_fails=3 fail_timeout=30s;
    server filer2:8888 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name filer.example.com;

    location / {
        proxy_pass http://filer_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Увеличенные таймауты для больших файлов
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
        
        # Увеличенный размер буфера
        client_max_body_size 10G;
    }
}
```

#### Load Balancer для S3

```nginx
upstream s3_backend {
    least_conn;
    server s3-gw1:8333 max_fails=3 fail_timeout=30s;
    server s3-gw2:8333 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name s3.example.com *.s3.example.com;

    location / {
        proxy_pass http://s3_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # S3-специфичные заголовки
        proxy_set_header Authorization $http_authorization;
        proxy_pass_header Authorization;
        
        client_max_body_size 10G;
        proxy_request_buffering off;
    }
}
```

---

## Репликация и отказоустойчивость

### Схемы репликации

SeaweedFS использует синхронную репликацию на уровне записи. Формат: `XYZ`

#### Примеры схем

**000 - Без репликации**
- Только для тестирования
- Потеря данных при сбое диска

**001 - Same Rack Replication**
```
Rack1
├── Server1 (Primary)
└── Server2 (Replica)
```
- Защита от сбоя сервера
- Не защищает от сбоя rack

**010 - Different Rack Replication**
```
Rack1
└── Server1 (Primary)

Rack2
└── Server2 (Replica)
```
- Защита от сбоя rack
- Рекомендуется для single DC

**100 - Different Data Center**
```
DC1
└── Server1 (Primary)

DC2
└── Server2 (Replica)
```
- Защита от сбоя DC
- Увеличенная latency

**200 - Two Data Centers**
```
DC1
└── Server1 (Primary)

DC2
└── Server2 (Replica)

DC3
└── Server3 (Replica)
```
- Максимальная защита
- 3 копии данных

### Настройка репликации для collections

```bash
# Через weed shell
weed shell

# Создание collection с конкретной схемой репликации
> collection.create -name=critical-data -replication=200

# Создание collection с TTL
> collection.create -name=temp-data -replication=001 -ttl=7d

# Создание collection для конкретного типа дисков
> collection.create -name=hot-data -replication=010 -diskType=ssd
```

### Автоматическая репликация

При записи файла репликация происходит автоматически:

1. Клиент получает file ID от Master
2. Master возвращает primary volume и replica volumes
3. Клиент пишет в primary
4. Primary автоматически реплицирует на replicas
5. Запись считается успешной после подтверждения всех реплик

### Проверка статуса репликации

```bash
# Проверка топологии и реплик
curl "http://master1:9333/vol/status" | jq

# Проверка конкретного volume
curl "http://master1:9333/vol/lookup?volumeId=5" | jq

# Пример ответа
{
  "volumeId": "5",
  "locations": [
    {
      "url": "volume1:8080",
      "publicUrl": "volume1.example.com:8080"
    },
    {
      "url": "volume2:8080",
      "publicUrl": "volume2.example.com:8080"
    }
  ]
}
```

### Восстановление после сбоя

#### Сценарий 1: Сбой одного Volume сервера

Автоматическое поведение:
- Master помечает сервер как недоступный
- Клиенты автоматически переключаются на реплики
- После восстановления сервер синхронизируется автоматически

Ручная проверка:

```bash
# Проверить недоступные серверы
curl "http://master1:9333/vol/status" | jq '.Volumes.DataCenters[].Racks[].DataNodes[] | select(.Free == 0)'

# Вывести volumes, требующие внимания
weed shell
> volume.list -readonly
```

#### Сценарий 2: Полная потеря Volume сервера

```bash
# 1. Подготовить новый сервер с тем же именем/IP
# 2. Восстановить volumes из реплик

weed shell

# Копировать volume с другого сервера
> volume.copy -volumeId=5 -source=volume2:8080 -target=volume1:8080

# Или восстановить все volumes для сервера
> volume.fix.replication -n
```

#### Сценарий 3: Сбой Master сервера

При использовании Raft кластера:
- Автоматическое переизбрание leader
- Downtime < 10 секунд
- Данные не теряются

```bash
# Проверить статус кластера
curl "http://master1:9333/cluster/status"

# Если master не восстановился, перезапустить
sudo systemctl restart seaweedfs-master
```

### Rack и Data Center Awareness

#### Настройка топологии

```bash
# Volume сервер в DC1, Rack1
weed volume \
  -dataCenter=dc1 \
  -rack=rack1 \
  -mserver=master1:9333

# Volume сервер в DC1, Rack2
weed volume \
  -dataCenter=dc1 \
  -rack=rack2 \
  -mserver=master1:9333

# Volume сервер в DC2, Rack1
weed volume \
  -dataCenter=dc2 \
  -rack=rack1 \
  -mserver=master1:9333
```

#### Проверка топологии

```bash
curl "http://master1:9333/dir/status" | jq

# Пример структуры
{
  "Topology": {
    "DataCenters": [
      {
        "Id": "dc1",
        "Racks": [
          {"Id": "rack1", "DataNodeCount": 3},
          {"Id": "rack2", "DataNodeCount": 3}
        ]
      },
      {
        "Id": "dc2",
        "Racks": [
          {"Id": "rack1", "DataNodeCount": 3}
        ]
      }
    ]
  }
}
```

### Мониторинг отказоустойчивости

#### Healthcheck скрипт

```bash
#!/bin/bash
# /usr/local/bin/seaweed-health-check.sh

MASTER="master1:9333"
ALERT_EMAIL="ops@example.com"

# Проверка Master
if ! curl -sf "http://${MASTER}/cluster/healthz" > /dev/null; then
    echo "Master cluster unhealthy" | mail -s "SeaweedFS Alert" ${ALERT_EMAIL}
fi

# Проверка Volume серверов
UNHEALTHY=$(curl -s "http://${MASTER}/vol/status" | jq -r '.Volumes.DataCenters[].Racks[].DataNodes[] | select(.MaxVolumeCount == 0) | .Url')

if [ -n "$UNHEALTHY" ]; then
    echo "Unhealthy volume servers: ${UNHEALTHY}" | mail -s "SeaweedFS Alert" ${ALERT_EMAIL}
fi

# Проверка under-replicated volumes
UNDER_REPLICATED=$(weed shell -master="${MASTER}" <<EOF | grep "under"
volume.fix.replication -n
EOF
)

if [ -n "$UNDER_REPLICATED" ]; then
    echo "Under-replicated volumes detected" | mail -s "SeaweedFS Alert" ${ALERT_EMAIL}
fi
```

```bash
# Добавить в cron
crontab -e
*/5 * * * * /usr/local/bin/seaweed-health-check.sh
```

---

## Балансировка нагрузки

### HAProxy конфигурация

#### Для Filer

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 20000
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

# Filer backend
frontend filer_frontend
    bind *:80
    mode http
    default_backend filer_backend
    
    # Health check endpoint
    acl health_check path_beg /health
    use_backend health_backend if health_check

backend filer_backend
    mode http
    balance leastconn
    option httpchk GET /
    
    server filer1 filer1:8888 check inter 5s fall 3 rise 2
    server filer2 filer2:8888 check inter 5s fall 3 rise 2
    server filer3 filer3:8888 check inter 5s fall 3 rise 2

backend health_backend
    mode http
    server health 127.0.0.1:8080
```

#### Для S3 Gateway

```haproxy
# S3 frontend
frontend s3_frontend
    bind *:8333
    mode http
    default_backend s3_backend
    
    # Логирование S3 запросов
    log-format "%ci:%cp [%t] %ft %b/%s %Tq/%Tw/%Tc/%Tr/%Tt %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %hr %hs %{+Q}r"

backend s3_backend
    mode http
    balance leastconn
    option httpchk GET /
    
    # Sticky sessions для multipart uploads
    stick-table type string len 32 size 100k expire 1h
    stick on url_param(uploadId)
    
    server s3-gw1 s3-gw1:8333 check inter 5s fall 3 rise 2
    server s3-gw2 s3-gw2:8333 check inter 5s fall 3 rise 2
    server s3-gw3 s3-gw3:8333 check inter 5s fall 3 rise 2

# Статистика
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if LOCALHOST
```

### Nginx конфигурация

#### Для Filer с кешированием

```nginx
# /etc/nginx/conf.d/seaweedfs-filer.conf

proxy_cache_path /var/cache/nginx/seaweedfs 
    levels=1:2 
    keys_zone=seaweedfs_cache:100m 
    max_size=10g 
    inactive=60m 
    use_temp_path=off;

upstream filer_backend {
    least_conn;
    
    server filer1:8888 max_fails=3 fail_timeout=30s;
    server filer2:8888 max_fails=3 fail_timeout=30s;
    server filer3:8888 max_fails=3 fail_timeout=30s;
    
    keepalive 32;
}

server {
    listen 80;
    server_name filer.example.com;
    
    # Логирование
    access_log /var/log/nginx/filer-access.log combined;
    error_log /var/log/nginx/filer-error.log;
    
    # Большие файлы
    client_max_body_size 10G;
    client_body_buffer_size 128k;
    
    # Таймауты
    proxy_connect_timeout 600;
    proxy_send_timeout 600;
    proxy_read_timeout 600;
    send_timeout 600;
    
    location / {
        proxy_pass http://filer_backend;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Keep-alive
        proxy_set_header Connection "";
        
        # Кеширование GET запросов
        proxy_cache seaweedfs_cache;
        proxy_cache_valid 200 60m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;
        
        add_header X-Cache-Status $upstream_cache_status;
    }
    
    # Отключить кеш для POST/PUT/DELETE
    location ~* ^.+\.(POST|PUT|DELETE)$ {
        proxy_pass http://filer_backend;
        proxy_cache off;
    }
}
```

#### Для S3 Gateway

```nginx
# /etc/nginx/conf.d/seaweedfs-s3.conf

upstream s3_backend {
    least_conn;
    
    server s3-gw1:8333 max_fails=3 fail_timeout=30s;
    server s3-gw2:8333 max_fails=3 fail_timeout=30s;
    
    keepalive 32;
}

server {
    listen 80;
    server_name s3.example.com *.s3.example.com;
    
    client_max_body_size 10G;
    
    # Отключить buffering для streaming
    proxy_request_buffering off;
    proxy_buffering off;
    
    location / {
        proxy_pass http://s3_backend;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # S3 authentication headers
        proxy_set_header Authorization $http_authorization;
        proxy_pass_header Authorization;
        proxy_pass_header Date;
        proxy_pass_header Server;
        
        # Keep-alive
        proxy_set_header Connection "";
        
        # Таймауты для больших файлов
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
    }
}
```

### DNS Round Robin

Простейший метод балансировки:

```bash
# /etc/hosts или DNS записи
10.0.1.10 filer.example.com
10.0.1.11 filer.example.com
10.0.1.12 filer.example.com
```

Клиенты автоматически распределяются по серверам через DNS resolution.

### Client-side балансировка

Многие клиенты поддерживают множественные endpoints:

```python
# Python пример с boto3
import boto3
from botocore.config import Config

# Несколько endpoints
s3 = boto3.client(
    's3',
    endpoint_url='http://s3-gw1:8333',  # Primary
    aws_access_key_id='admin_access_key',
    aws_secret_access_key='admin_secret_key',
    config=Config(
        retries={'max_attempts': 3},
        connect_timeout=5,
        read_timeout=60
    )
)

# Fallback на другой gateway при ошибке
try:
    s3.get_object(Bucket='bucket', Key='key')
except Exception:
    s3 = boto3.client(
        's3',
        endpoint_url='http://s3-gw2:8333',  # Fallback
        aws_access_key_id='admin_access_key',
        aws_secret_access_key='admin_secret_key'
    )
```

---

## Мониторинг и алертинг

### Prometheus метрики

SeaweedFS экспортирует метрики в формате Prometheus на отдельных портах.

#### Конфигурация Prometheus

```yaml
# /etc/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Master servers
  - job_name: 'seaweedfs-master'
    static_configs:
      - targets:
          - 'master1:9324'
          - 'master2:9324'
          - 'master3:9324'
        labels:
          cluster: 'production'
          role: 'master'
  
  # Volume servers
  - job_name: 'seaweedfs-volume'
    static_configs:
      - targets:
          - 'volume1:9325'
          - 'volume2:9325'
          - 'volume3:9325'
        labels:
          cluster: 'production'
          role: 'volume'
  
  # Filer servers
  - job_name: 'seaweedfs-filer'
    static_configs:
      - targets:
          - 'filer1:9326'
          - 'filer2:9326'
        labels:
          cluster: 'production'
          role: 'filer'
  
  # S3 gateways
  - job_name: 'seaweedfs-s3'
    static_configs:
      - targets:
          - 's3-gw1:9327'
          - 's3-gw2:9327'
        labels:
          cluster: 'production'
          role: 's3'
```

### Ключевые метрики

#### Master метрики

```promql
# Количество volume серверов
weed_master_volume_servers_count

# Общее количество volumes
weed_master_volumes_count

# Количество файлов
weed_master_file_count

# Используемое пространство
weed_master_bytes_total

# Latency операций
weed_master_request_duration_seconds

# Состояние Raft cluster
weed_master_is_leader
```

#### Volume метрики

```promql
# Доступное пространство
weed_volume_disk_free_bytes

# Использованное пространство
weed_volume_disk_used_bytes

# IOPS
rate(weed_volume_request_total[5m])

# Latency чтения
weed_volume_read_request_duration_seconds

# Latency записи
weed_volume_write_request_duration_seconds

# Ошибки
rate(weed_volume_request_errors_total[5m])
```

#### Filer метрики

```promql
# Количество запросов
rate(weed_filer_request_total[5m])

# Latency
weed_filer_request_duration_seconds

# Размер загружаемых файлов
weed_filer_upload_bytes_total

# Ошибки
rate(weed_filer_request_errors_total[5m])
```

### Grafana дашборды

#### Импорт готового дашборда

```bash
# Скачать дашборд из репозитория SeaweedFS
wget https://raw.githubusercontent.com/seaweedfs/seaweedfs/master/docker/grafana.json

# Импортировать в Grafana через UI
# Dashboard -> Import -> Upload JSON
```

#### Создание кастомного дашборда

Основные панели:

**Cluster Overview:**
```promql
# Total storage capacity
sum(weed_volume_disk_total_bytes)

# Used storage
sum(weed_volume_disk_used_bytes)

# Free storage
sum(weed_volume_disk_free_bytes)

# Total files
sum(weed_master_file_count)

# Volume servers online
count(weed_volume_disk_free_bytes)
```

**Performance:**
```promql
# Read IOPS
sum(rate(weed_volume_read_request_total[5m]))

# Write IOPS
sum(rate(weed_volume_write_request_total[5m]))

# Read throughput
sum(rate(weed_volume_read_bytes_total[5m]))

# Write throughput
sum(rate(weed_volume_write_bytes_total[5m]))

# 95th percentile latency
histogram_quantile(0.95, 
  rate(weed_volume_request_duration_seconds_bucket[5m])
)
```

**Errors:**
```promql
# Error rate
sum(rate(weed_volume_request_errors_total[5m]))

# Error ratio
sum(rate(weed_volume_request_errors_total[5m])) / 
sum(rate(weed_volume_request_total[5m]))
```

### Alertmanager правила

```yaml
# /etc/prometheus/alerts.yml

groups:
  - name: seaweedfs
    interval: 30s
    rules:
      # Master недоступен
      - alert: MasterDown
        expr: up{job="seaweedfs-master"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "SeaweedFS Master {{ $labels.instance }} is down"
          description: "Master server has been down for more than 1 minute"
      
      # Нет leader в Raft cluster
      - alert: NoRaftLeader
        expr: sum(weed_master_is_leader) == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "No Raft leader in cluster"
          description: "Master cluster has no leader"
      
      # Volume сервер недоступен
      - alert: VolumeServerDown
        expr: up{job="seaweedfs-volume"} == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Volume server {{ $labels.instance }} is down"
      
      # Заканчивается место
      - alert: DiskSpaceLow
        expr: |
          (weed_volume_disk_free_bytes / weed_volume_disk_total_bytes) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Less than 10% free space remaining"
      
      # Критически мало места
      - alert: DiskSpaceCritical
        expr: |
          (weed_volume_disk_free_bytes / weed_volume_disk_total_bytes) < 0.05
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Disk space critical on {{ $labels.instance }}"
          description: "Less than 5% free space remaining"
      
      # Высокая latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            rate(weed_volume_request_duration_seconds_bucket[5m])
          ) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.instance }}"
          description: "95th percentile latency is above 1 second"
      
      # Высокий процент ошибок
      - alert: HighErrorRate
        expr: |
          sum(rate(weed_volume_request_errors_total[5m])) /
          sum(rate(weed_volume_request_total[5m])) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is above 5%"
```

### Логирование

#### Centralized logging с ELK

**Filebeat конфигурация:**

```yaml
# /etc/filebeat/filebeat.yml

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/seaweedfs/*.log
    fields:
      service: seaweedfs
      environment: production
    multiline:
      pattern: '^\d{4}-\d{2}-\d{2}'
      negate: true
      match: after

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "seaweedfs-%{+yyyy.MM.dd}"

setup.kibana:
  host: "kibana:5601"
```

#### Journald логи

```bash
# Просмотр логов Master
sudo journalctl -u seaweedfs-master -f

# Просмотр логов Volume
sudo journalctl -u seaweedfs-volume -f --since "1 hour ago"

# Поиск ошибок
sudo journalctl -u seaweedfs-volume -p err -n 100

# Экспорт в файл
sudo journalctl -u seaweedfs-master --since "2024-01-01" > master-logs.txt
```

### Healthcheck endpoints

```bash
# Master health
curl "http://master1:9333/cluster/healthz"

# Volume health
curl "http://volume1:8080/status"

# Filer health
curl "http://filer1:8888/"

# S3 health
curl "http://s3-gw1:8333/"
```

### Custom monitoring скрипты

```bash
#!/bin/bash
# /usr/local/bin/seaweed-monitor.sh

MASTER="master1:9333"
PUSHGATEWAY="pushgateway:9091"
JOB="seaweedfs_custom"

# Проверка under-replicated volumes
UNDER_REPL=$(weed shell -master="${MASTER}" <<EOF | grep -c "under"
volume.fix.replication -n
EOF
)

echo "seaweedfs_under_replicated_volumes ${UNDER_REPL}" | curl --data-binary @- \
  "http://${PUSHGATEWAY}/metrics/job/${JOB}/instance/$(hostname)"

# Проверка свободного места по data centers
for DC in $(curl -s "http://${MASTER}/dir/status" | jq -r '.Topology.DataCenters[].Id'); do
    FREE=$(curl -s "http://${MASTER}/dir/status" | \
           jq ".Topology.DataCenters[] | select(.Id==\"${DC}\") | .Free")
    
    echo "seaweedfs_dc_free_volumes{dc=\"${DC}\"} ${FREE}" | \
      curl --data-binary @- "http://${PUSHGATEWAY}/metrics/job/${JOB}/instance/$(hostname)"
done
```

```bash
# Добавить в cron
crontab -e
*/5 * * * * /usr/local/bin/seaweed-monitor.sh
```

---

## Backup и восстановление

### Стратегии резервного копирования

#### Уровень 1: Репликация (встроенная)