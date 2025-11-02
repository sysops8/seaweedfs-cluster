# Production-ready SeaweedFS: Полное руководство по установке и настройке

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

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT APPLICATIONS                      │
│              (S3 API / HTTP API / Mount)                     │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐       ┌────────▼───────┐
│  Filer Servers │◄─────►│  S3 Gateway    │
│   (Metadata)   │       │   (S3 API)     │
└───────┬────────┘       └────────────────┘
        │
        │ (Metadata Storage)
        ▼
┌────────────────┐
│   Database     │
│ (PostgreSQL/   │
│  MySQL/Redis/  │
│   LevelDB)     │
└────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Master Servers                          │
│              (Leader Election / Topology)                    │
│         master-1:9333  master-2:9333  master-3:9333         │
└────────────────────┬────────────────────────────────────────┘
                     │
          ┌──────────┼──────────┐
          │          │          │
┌─────────▼────┐ ┌──▼──────────▼┐ ┌─────────────┐
│ Volume Rack1 │ │ Volume Rack2 │ │ Volume Rack3│
│ volume-1-1   │ │ volume-2-1   │ │ volume-3-1  │
│ volume-1-2   │ │ volume-2-2   │ │ volume-3-2  │
│ volume-1-3   │ │ volume-2-3   │ │ volume-3-3  │
└──────────────┘ └──────────────┘ └─────────────┘
   (Data Files)     (Data Files)     (Data Files)
```

### Как работает хранение файлов

**Суперблоки (Volume Files)**

Вместо хранения миллионов отдельных файлов, SeaweedFS упаковывает их в большие "суперблоки". Это решает проблему ограничений файловых систем:

```
Обычная ФС:
file1.jpg (inode 1001)
file2.jpg (inode 1002)
...
file1000000.jpg (inode 1001000) ← Проблемы с производительностью

SeaweedFS:
volume_001.dat (содержит file1.jpg, file2.jpg, ..., file500000.jpg)
volume_002.dat (содержит file500001.jpg, ..., file1000000.jpg)
```

**Путь записи файла**

1. Клиент запрашивает у Master сервера место для записи
2. Master возвращает volume ID и адреса volume серверов
3. Клиент пишет данные напрямую на volume серверы (в 2+ реплики)
4. Volume серверы реплицируют данные между собой
5. После успешной записи Filer сохраняет метаданные в базу данных
6. Клиенту возвращается подтверждение

**Путь чтения файла**

1. Клиент запрашивает файл через Filer
2. Filer ищет метаданные в базе данных
3. Filer получает topology от Master сервера
4. Клиент перенаправляется напрямую к Volume серверу
5. Volume сервер отдает данные из суперблока

---

## Почему SeaweedFS

### Сравнение с альтернативами

| Характеристика | MinIO | Ceph | SeaweedFS |
|---------------|-------|------|-----------|
| Хранение метаданных | Вместе с данными | Вместе с данными | Отдельная БД |
| Масштабирование мелких файлов | Деградирует | Деградирует | Без деградации |
| Требования к железу | Одинаковые диски | Высокие | Низкие, любые диски |
| Ребалансировка при добавлении дисков | Требуется | Требуется | Не требуется |
| Сложность настройки | Низкая | Высокая | Средняя |
| Репликация в реальном времени | Нет (только EC) | Да | Да |
| S3 API | Полная | Через RGW | Полная |

### Когда выбирать SeaweedFS

**SeaweedFS подходит если:**
- Нужно хранить сотни миллионов файлов
- Файлы разного размера (от килобайт до гигабайт)
- Важна скорость доступа без деградации при росте
- Бюджет ограничен, нельзя использовать одинаковые диски
- Нужна простота эксплуатации
- Требуется синхронная репликация с гарантиями

**MinIO подходит если:**
- Небольшое количество больших файлов
- Есть возможность использовать одинаковые диски
- Erasure Coding достаточен для защиты данных
- Нужна максимальная производительность на запись
- Главный приоритет — совместимость с AWS S3

**Ceph подходит если:**
- Нужны блочные устройства и объектное хранилище одновременно
- Требуется максимальная отказоустойчивость
- Есть команда с экспертизой в Ceph
- Хорошая сетевая инфраструктура (10G+)

---

## Требования к окружению

### Аппаратные требования

**Master серверы (минимум 3 ноды для production)**

- **CPU**: 2-4 ядра
- **RAM**: 4-8 GB
- **Диск**: 50-100 GB SSD для метаданных
- **Сеть**: 1 Gbit/s
- **Количество**: 3, 5 или 7 (нечетное для Raft консенсуса)

**Volume серверы (рекомендуется минимум 6 нод)**

- **CPU**: 8-16 ядер
- **RAM**: 16-64 GB (4GB на 100 volumes)
- **Диск**: 4-24× HDD/SSD, от 1 TB каждый
- **Сеть**: 10 Gbit/s (рекомендуется)
- **Файловая система**: XFS (рекомендуется) или EXT4

**Filer серверы (минимум 2 ноды)**

- **CPU**: 4-8 ядер
- **RAM**: 8-32 GB
- **Диск**: 100-500 GB SSD
- **Сеть**: 1-10 Gbit/s
- **База данных**: отдельный кластер PostgreSQL/MySQL или встроенный LevelDB

### Программные требования

**Операционная система:**
- Ubuntu 20.04+ / Debian 10+
- CentOS 7+ / Rocky Linux 8+
- RHEL 7+

**Дополнительное ПО:**
- systemd (для управления сервисами)
- firewalld / iptables (настройка firewall)
- Prometheus + Grafana (мониторинг)
- Nginx / HAProxy (балансировка нагрузки)

### Сетевая топология

**Порты для firewall:**

| Компонент | Порт | Протокол | Назначение |
|-----------|------|----------|------------|
| Master | 9333 | gRPC | Основной API |
| Master | 19333 | HTTP | Метрики |
| Volume | 8080 | HTTP | Загрузка/скачивание файлов |
| Volume | 18080 | gRPC | Репликация |
| Filer | 8888 | HTTP | Filer API |
| Filer | 18888 | gRPC | Filer gRPC |
| S3 Gateway | 8333 | HTTP | S3 API |

**Пример правил firewall (firewalld):**

```bash
# Master серверы
firewall-cmd --permanent --add-port=9333/tcp
firewall-cmd --permanent --add-port=19333/tcp

# Volume серверы
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-port=18080/tcp

# Filer серверы
firewall-cmd --permanent --add-port=8888/tcp
firewall-cmd --permanent --add-port=18888/tcp
firewall-cmd --permanent --add-port=8333/tcp

firewall-cmd --reload
```

---

## Планирование кластера

### Расчет capacity

**Формула для расчета полезного объема:**

```
Usable Space = (N_servers × Disks_per_server × Disk_size) / Replication_factor
```

**Примеры конфигураций:**

**Конфигурация 1: Малый кластер**
- 6 volume серверов × 4 диска × 4 TB = 96 TB raw
- Репликация 2× (rack awareness): 96 / 2 = **48 TB usable**
- Подходит для: 100-200 млн файлов

**Конфигурация 2: Средний кластер**
- 9 volume серверов × 8 дисков × 6 TB = 432 TB raw
- Репликация 3× (DC awareness): 432 / 3 = **144 TB usable**
- Подходит для: 300-500 млн файлов

**Конфигурация 3: Большой кластер**
- 15 volume серверов × 12 дисков × 10 TB = 1.8 PB raw
- Репликация 2× (rack awareness): 1800 / 2 = **900 TB usable**
- Подходит для: 1+ млрд файлов

### Топология production кластера

**Рекомендуемая конфигурация для production:**

```
Датацентр DC1
├── Rack 1
│   ├── master-1 (leader)
│   ├── volume-1-1, volume-1-2, volume-1-3
│   └── filer-1
├── Rack 2
│   ├── master-2 (follower)
│   ├── volume-2-1, volume-2-2, volume-2-3
│   └── filer-2
└── Rack 3
    ├── master-3 (follower)
    ├── volume-3-1, volume-3-2, volume-3-3
    └── filer-3

Балансировщики:
- nginx-lb-1 (master API)
- nginx-lb-2 (filer/S3 API)
```

**Для multi-DC конфигурации:**

```
DC1 (основной)           DC2 (резервный)
├── 3 master            ├── 2 master
├── 6 volume            ├── 6 volume
└── 2 filer             └── 2 filer
```

---

## Установка Master серверов

### Скачивание и установка бинарного файла

```bash
# Создание пользователя
sudo useradd -r -s /bin/false seaweedfs

# Скачивание последней версии
cd /tmp
LATEST_VERSION=$(curl -s https://api.github.com/repos/seaweedfs/seaweedfs/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
wget https://github.com/seaweedfs/seaweedfs/releases/download/${LATEST_VERSION}/linux_amd64.tar.gz

# Установка
sudo tar -xzf linux_amd64.tar.gz -C /usr/local/bin/
sudo chmod +x /usr/local/bin/weed
sudo chown root:root /usr/local/bin/weed

# Проверка версии
weed version
```

### Создание директорий

```bash
# Создание структуры каталогов
sudo mkdir -p /data/seaweedfs/metadata
sudo mkdir -p /etc/seaweedfs
sudo mkdir -p /var/log/seaweedfs

# Установка прав доступа
sudo chown -R seaweedfs:seaweedfs /data/seaweedfs
sudo chown -R seaweedfs:seaweedfs /var/log/seaweedfs
sudo chmod 750 /data/seaweedfs/metadata
```

### Конфигурация systemd для Master

Создайте файл `/etc/systemd/system/seaweedfs-master.service`:

```ini
[Unit]
Description=SeaweedFS Master Server
Documentation=https://github.com/seaweedfs/seaweedfs
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs

# Основные параметры
ExecStart=/usr/local/bin/weed master \
  -ip=master-1 \
  -port=9333 \
  -mdir=/data/seaweedfs/metadata \
  -peers=master-1:9333,master-2:9333,master-3:9333 \
  -defaultReplication=010 \
  -volumeSizeLimitMB=30000 \
  -volumePreallocate \
  -garbageThreshold=0.3 \
  -pulseSeconds=5 \
  -metricsAddress=:19333

# Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=seaweedfs-master

# Автоматический рестарт
Restart=always
RestartSec=30
StartLimitBurst=5
StartLimitInterval=300

# Ограничения ресурсов
LimitNOFILE=65536
LimitNPROC=32768

# Безопасность
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/data/seaweedfs

[Install]
WantedBy=multi-user.target
```

**Пояснение параметров:**

- `-ip` — IP адрес или hostname текущего master сервера
- `-port` — порт для gRPC API (по умолчанию 9333)
- `-mdir` — директория для метаданных master сервера
- `-peers` — список всех master серверов в кластере
- `-defaultReplication` — схема репликации по умолчанию (010 = разные rack)
- `-volumeSizeLimitMB` — максимальный размер volume файла (30GB)
- `-volumePreallocate` — предварительное выделение места для volume
- `-garbageThreshold` — порог для сборки мусора (30%)
- `-pulseSeconds` — интервал heartbeat от volume серверов
- `-metricsAddress` — адрес для Prometheus метрик

### Конфигурационный файл Master

Создайте `/etc/seaweedfs/master.toml`:

```toml
# Master Server Configuration

[master]
# Базовые настройки
ip = "master-1"
port = 9333
peers = ["master-1:9333", "master-2:9333", "master-3:9333"]

# Директория метаданных
mdir = "/data/seaweedfs/metadata"

# Репликация
defaultReplication = "010"
volumeSizeLimitMB = 30000
volumePreallocate = true

# Сборка мусора
garbageThreshold = 0.3
pulseSeconds = 5

# Метрики
[master.metrics]
address = ":19333"
intervalSeconds = 15

# Безопасность
[master.security]
# Whitelist для доступа к API
allowed_origins = ["10.0.0.0/8", "192.168.0.0/16"]
```

### Запуск Master кластера

**На каждом master сервере выполните:**

```bash
# Загрузка конфигурации systemd
sudo systemctl daemon-reload

# Включение автозапуска
sudo systemctl enable seaweedfs-master

# Запуск сервиса
sudo systemctl start seaweedfs-master

# Проверка статуса
sudo systemctl status seaweedfs-master

# Проверка логов
sudo journalctl -u seaweedfs-master -f
```

### Проверка Master кластера

```bash
# Проверка статуса кластера
curl http://master-1:9333/cluster/status?pretty=y

# Проверка Raft консенсуса
curl http://master-1:9333/cluster/raft?pretty=y

# Проверка topology
curl http://master-1:9333/dir/status?pretty=y

# Через weed shell
weed shell -master=master-1:9333 << EOF
cluster.ps
volume.list
EOF
```

**Ожидаемый вывод статуса кластера:**

```json
{
  "IsLeader": true,
  "Leader": "master-1:9333",
  "Peers": [
    "master-1:9333",
    "master-2:9333",
    "master-3:9333"
  ]
}
```

### Health check скрипт для Master

Создайте `/usr/local/bin/check-master-health.sh`:

```bash
#!/bin/bash

MASTER_HOST="localhost"
MASTER_PORT="9333"

# Проверка доступности API
if ! curl -sf "http://${MASTER_HOST}:${MASTER_PORT}/cluster/status" > /dev/null; then
    echo "ERROR: Master API не отвечает"
    exit 1
fi

# Проверка наличия лидера
LEADER=$(curl -s "http://${MASTER_HOST}:${MASTER_PORT}/cluster/status" | jq -r '.Leader')
if [ -z "$LEADER" ] || [ "$LEADER" = "null" ]; then
    echo "ERROR: Нет лидера в кластере"
    exit 1
fi

echo "OK: Master работает, лидер: $LEADER"
exit 0
```

```bash
chmod +x /usr/local/bin/check-master-health.sh
```

---

## Установка Volume серверов

### Подготовка дисков

```bash
# Форматирование дисков в XFS (рекомендуется)
sudo mkfs.xfs -f -i size=512 /dev/sdb
sudo mkfs.xfs -f -i size=512 /dev/sdc
sudo mkfs.xfs -f -i size=512 /dev/sdd
sudo mkfs.xfs -f -i size=512 /dev/sde

# Создание точек монтирования
sudo mkdir -p /data/disk{1,2,3,4}/seaweedfs/volumes

# Получение UUID дисков
sudo blkid /dev/sdb
sudo blkid /dev/sdc
sudo blkid /dev/sdd
sudo blkid /dev/sde

# Добавление в /etc/fstab
cat <<EOF | sudo tee -a /etc/fstab
UUID=xxx-xxx-xxx /data/disk1 xfs noatime,nodiratime,logbufs=8,logbsize=256k 0 2
UUID=yyy-yyy-yyy /data/disk2 xfs noatime,nodiratime,logbufs=8,logbsize=256k 0 2
UUID=zzz-zzz-zzz /data/disk3 xfs noatime,nodiratime,logbufs=8,logbsize=256k 0 2
UUID=www-www-www /data/disk4 xfs noatime,nodiratime,logbufs=8,logbsize=256k 0 2
EOF

# Монтирование
sudo mount -a

# Установка прав
sudo chown -R seaweedfs:seaweedfs /data/disk{1,2,3,4}
```

### Конфигурация systemd для Volume

Создайте `/etc/systemd/system/seaweedfs-volume.service`:

```ini
[Unit]
Description=SeaweedFS Volume Server
Documentation=https://github.com/seaweedfs/seaweedfs
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs

# Основные параметры
ExecStart=/usr/local/bin/weed volume \
  -ip=volume-1-1 \
  -port=8080 \
  -mserver=master-1:9333,master-2:9333,master-3:9333 \
  -dir=/data/disk1/seaweedfs/volumes,/data/disk2/seaweedfs/volumes,/data/disk3/seaweedfs/volumes,/data/disk4/seaweedfs/volumes \
  -max=25 \
  -rack=rack-1 \
  -dataCenter=dc1 \
  -index=leveldb \
  -compactionMBps=100 \
  -fileSizeLimitMB=1024 \
  -minFreeSpacePercent=5 \
  -metricsAddress=:18080

# Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=seaweedfs-volume

# Автоматический рестарт
Restart=always
RestartSec=30
StartLimitBurst=5
StartLimitInterval=300

# Ограничения ресурсов
LimitNOFILE=65536
LimitNPROC=32768

# Безопасность
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/data/disk1 /data/disk2 /data/disk3 /data/disk4

[Install]
WantedBy=multi-user.target
```

**Пояснение параметров:**

- `-ip` — IP адрес volume сервера
- `-port` — HTTP порт для API
- `-mserver` — список master серверов
- `-dir` — список директорий для volume (можно указать несколько дисков через запятую)
- `-max` — максимальное количество volumes на одну директорию
- `-rack` — идентификатор rack для rack awareness
- `-dataCenter` — идентификатор датацентра
- `-index=leveldb` — тип индекса (leveldb быстрее чем btree)
- `-compactionMBps` — лимит скорости компрессии (MB/s)
- `-fileSizeLimitMB` — максимальный размер одного файла
- `-minFreeSpacePercent` — минимальный процент свободного места

### Конфигурация для разных rack

**Rack 1 (volume-1-{1,2,3}):**

```ini
ExecStart=/usr/local/bin/weed volume \
  -ip=volume-1-1 \
  -rack=rack-1 \
  -dataCenter=dc1 \
  ...
```

**Rack 2 (volume-2-{1,2,3}):**

```ini
ExecStart=/usr/local/bin/weed volume \
  -ip=volume-2-1 \
  -rack=rack-2 \
  -dataCenter=dc1 \
  ...
```

**Rack 3 (volume-3-{1,2,3}):**

```ini
ExecStart=/usr/local/bin/weed volume \
  -ip=volume-3-1 \
  -rack=rack-3 \
  -dataCenter=dc1 \
  ...
```

### Запуск Volume серверов

```bash
# На каждом volume сервере
sudo systemctl daemon-reload
sudo systemctl enable seaweedfs-volume
sudo systemctl start seaweedfs-volume

# Проверка статуса
sudo systemctl status seaweedfs-volume

# Проверка доступности
curl http://volume-1-1:8080/status?pretty=y

# Проверка логов
sudo journalctl -u seaweedfs-volume -f
```

### Проверка Volume распределения

```bash
# Список всех volumes
weed shell -master=master-1:9333 << EOF
volume.list
EOF

# Проверка topology
curl http://master-1:9333/dir/status?pretty=y

# Статистика по rack
curl http://master-1:9333/cluster/status?pretty=y | jq '.Topology.DataCenters'
```

**Ожидаемый вывод:**

```json
{
  "Free": 12345,
  "Max": 15000,
  "DataCenters": [
    {
      "Free": 12345,
      "Max": 15000,
      "Racks": [
        {
          "DataNodes": [...]
        }
      ]
    }
  ]
}
```

---

## Настройка Filer и S3 Gateway

Filer предоставляет POSIX-like файловый интерфейс и S3-совместимый API. Метаданные хранятся в базе данных (PostgreSQL, MySQL, Redis или встроенный LevelDB).

### Выбор базы данных для метаданных

**Варианты хранения метаданных:**

| База данных | Производительность | Масштабируемость | Сложность | Рекомендация |
|-------------|-------------------|------------------|-----------|--------------|
| LevelDB | Высокая | Низкая | Низкая | Dev/тестирование |
| PostgreSQL | Средняя | Высокая | Средняя | **Production (рекомендуется)** |
| MySQL | Средняя | Высокая | Средняя | Production |
| Redis | Очень высокая | Средняя | Средняя | Горячие данные + PostgreSQL для холодных |

### Настройка PostgreSQL для Filer

```bash
# Установка PostgreSQL
sudo apt-get install postgresql-14 postgresql-client-14

# Создание базы данных и пользователя
sudo -u postgres psql << EOF
CREATE DATABASE seaweedfs;
CREATE USER seaweedfs WITH PASSWORD 'secure_password_here';
GRANT ALL PRIVILEGES ON DATABASE seaweedfs TO seaweedfs;
\c seaweedfs
GRANT ALL ON SCHEMA public TO seaweedfs;
EOF

# Создание таблицы для filer
sudo -u postgres psql -d seaweedfs << EOF
CREATE TABLE IF NOT EXISTS filemeta (
    dirhash   BIGINT,
    name      VARCHAR(65535),
    directory VARCHAR(65535),
    meta      BYTEA,
    PRIMARY KEY (dirhash, name)
);

CREATE INDEX IF NOT EXISTS idx_directory ON filemeta (directory);
CREATE INDEX IF NOT EXISTS idx_dirhash_name ON filemeta (dirhash, name);
EOF
```

### Конфигурация systemd для Filer

Создайте `/etc/systemd/system/seaweedfs-filer.service`:

```ini
[Unit]
Description=SeaweedFS Filer Server
Documentation=https://github.com/seaweedfs/seaweedfs
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple
User=seaweedfs
Group=seaweedfs

# Основные параметры
ExecStart=/usr/local/bin/weed filer \
  -ip=filer-1 \
  -port=8888 \
  -master=master-1:9333,master-2:9333,master-3:9333 \
  -dataCenter=dc1 \
  -defaultReplicaPlacement=010 \
  -disableHttp=false \
  -maxMB=0 \
  -metricsAddress=:18888

# Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=seaweedfs-filer

# Автоматический рестарт
Restart=always
RestartSec=30
StartLimitBurst=5
StartLimitInterval=300

# Ограничения ресурсов
LimitNOFILE=65536
LimitNPROC=32768

# Безопасность
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/data/seaweedfs

[Install]
WantedBy=multi-user.target
```

### Конфигурационный файл Filer

Создайте `/etc/seaweedfs/filer.toml`:

```toml
# Filer Configuration

[filer]
# Базовые настройки
ip = "filer-1"
port = 8888
master = "master-1:9333,master-2:9333,master-3:9333"

# Репликация по умолчанию
defaultReplicaPlacement = "010"

# Директория для локального кэша
dir = "/data/seaweedfs/filer"

# Размер файлов (0 = без ограничений)
maxMB = 0

# Метрики
[filer.metrics]
address = ":18888"

# PostgreSQL для метаданных
[postgres2]
enabled = true
createTable = true
hostname = "postgres-host"
port = 5432
username = "seaweedfs"
password = "secure_password_here"
database = "seaweedfs"
schema = "public"
sslmode = "disable"
connection_max_idle = 100
connection_max_open = 150
connection_max_lifetime_seconds = 0

# Опционально: Redis для кэширования
[redis_cluster2]
enabled = false
addresses = [
  "redis-1:6379",
  "redis-2:6379",
  "redis-3:6379"
]
password = ""
readOnly = false
```

### Настройка S3 Gateway

Для включения S3 API добавьте параметры в systemd service:

```ini
ExecStart=/usr/local/bin/weed filer \
  -ip=filer-1 \
  -port=8888 \
  -master=master-1:9333,master-2:9333,master-3:9333 \
  -s3 \
  -s3.port=8333 \
  -s3.domainName=s3.example.com \
  -s3.allowEmptyFolder=true \
  -s3.allowDeleteBucketNotEmpty=false \
  -s3.config=/etc/seaweedfs/s3.json \
  ...
```

### Конфигурация S3 credentials

Создайте `/etc/seaweedfs/s3.json`:

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
      "name": "app_user",
      "credentials": [
        {
          "accessKey": "app_access_key",
          "secretKey": "app_secret_key"
        }
      ],
      "actions": [
        "Read",
        "Write"
      ]
    },
    {
      "name": "readonly_user",
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

```bash
# Установка прав доступа
sudo chown seaweedfs:seaweedfs /etc/seaweedfs/s3.json
sudo chmod 600 /etc/seaweedfs/s3.json
```

### Запуск Filer

```bash
# На каждом filer сервере
sudo systemctl daemon-reload
sudo systemctl enable seaweedfs-filer
sudo systemctl start seaweedfs-filer

# Проверка статуса
sudo systemctl status seaweedfs-filer

# Проверка Filer API
curl http://filer-1:8888/?pretty=y

# Проверка S3 API
curl http://filer-1:8333/

# Проверка логов
sudo journalctl -u seaweedfs-filer -f
```

### Тестирование S3 API

**С помощью AWS CLI:**

```bash
# Установка AWS CLI
pip install awscli

# Конфигурация
aws configure set aws_access_key_id admin_access_key
aws configure set aws_secret_access_key admin_secret_key
aws configure set region us-east-1

# Создание bucket
aws s3 mb s3://test-bucket --endpoint-url http://filer-1:8333

# Загрузка файла
echo "Hello SeaweedFS" > test.txt
aws s3 cp test.txt s3://test-bucket/ --endpoint-url http://filer-1:8333

# Список файлов
aws s3 ls s3://test-bucket/ --endpoint-url http://filer-1:8333

# Скачивание файла
aws s3 cp s3://test-bucket/test.txt downloaded.txt --endpoint-url http://filer-1:8333

# Удаление файла
aws s3 rm s3://test-bucket/test.txt --endpoint-url http://filer-1:8333
```

**С помощью s3cmd:**

```bash
# Установка s3cmd
apt-get install s3cmd

# Конфигурация
cat > ~/.s3cfg << EOF
[default]
access_key = admin_access_key
secret_key = admin_secret_key
host_base = filer-1:8333
host_bucket = filer-1:8333
use_https = False
EOF

# Тестирование
s3cmd mb s3://test-bucket
s3cmd put test.txt s3://test-bucket/
s3cmd ls s3://test-bucket/
```

---

## Репликация и отказоустойчивость

### Схемы репликации

SeaweedFS использует гибкую систему репликации формата `XYZ`:

- **X** — количество реплик в других датацентрах
- **Y** — количество реплик в других rack в том же датацентре
- **Z** — количество реплик в том же rack

**Примеры схем:**

| Схема | Описание | Usable Space | Отказоустойчивость |
|-------|----------|--------------|-------------------|
| `000` | Нет репликации | 100% | Потеря одного диска = потеря данных |
| `001` | 2 копии в одном rack | 50% | Переживет потерю 1 сервера в rack |
| `010` | 2 копии в разных rack | 50% | Переживет потерю целого rack |
| `100` | 2 копии в разных DC | 50% | Переживет потерю целого датацентра |
| `200` | 3 копии в разных DC | 33% | Переживет потерю 2 датацентров |
| `020` | 3 копии в разных rack | 33% | Переживет потерю 2 rack |
| `002` | 3 копии в одном rack | 33% | Переживет потерю 2 серверов в rack |

### Настройка репликации

**Глобальная настройка (Master):**

```bash
# В systemd service или конфиге master
-defaultReplication=010
```

**На уровне коллекции:**

```bash
# Создание коллекции с определенной репликацией
weed shell -master=master-1:9333 << EOF
volume.create -replication=010 -collection=important-data
EOF

# Список коллекций
weed shell -master=master-1:9333 << EOF
collection.list
EOF
```

**При загрузке файла (через API):**

```bash
# Получение volume с определенной репликацией
curl "http://master-1:9333/dir/assign?replication=010"

# Ответ:
# {
#   "fid": "3,01637037d6",
#   "url": "volume-1-1:8080",
#   "publicUrl": "volume-1-1:8080",
#   "count": 1
# }

# Загрузка файла
curl -F "file=@photo.jpg" "http://volume-1-1:8080/3,01637037d6"
```

### Rack Awareness конфигурация

**Пример топологии с 3 rack:**

```
DataCenter: dc1
├── Rack: rack-1
│   ├── volume-1-1 (IP: 10.0.1.11)
│   ├── volume-1-2 (IP: 10.0.1.12)
│   └── volume-1-3 (IP: 10.0.1.13)
├── Rack: rack-2
│   ├── volume-2-1 (IP: 10.0.2.11)
│   ├── volume-2-2 (IP: 10.0.2.12)
│   └── volume-2-3 (IP: 10.0.2.13)
└── Rack: rack-3
    ├── volume-3-1 (IP: 10.0.3.11)
    ├── volume-3-2 (IP: 10.0.3.12)
    └── volume-3-3 (IP: 10.0.3.13)
```

**Конфигурация volume серверов:**

```bash
# Rack 1
weed volume -ip=10.0.1.11 -rack=rack-1 -dataCenter=dc1 ...
weed volume -ip=10.0.1.12 -rack=rack-1 -dataCenter=dc1 ...

# Rack 2
weed volume -ip=10.0.2.11 -rack=rack-2 -dataCenter=dc1 ...
weed volume -ip=10.0.2.12 -rack=rack-2 -dataCenter=dc1 ...

# Rack 3
weed volume -ip=10.0.3.11 -rack=rack-3 -dataCenter=dc1 ...
weed volume -ip=10.0.3.12 -rack=rack-3 -dataCenter=dc1 ...
```

**При репликации `010` (разные rack):**
- Основная копия: rack-1
- Реплика: rack-2 или rack-3
- При потере rack-1 данные доступны из другого rack

### Multi-DC конфигурация

**Пример топологии с 2 датацентрами:**

```
DataCenter: dc1 (основной)
├── master-1, master-2, master-3
├── volume-dc1-1, volume-dc1-2, volume-dc1-3
└── filer-1, filer-2

DataCenter: dc2 (резервный)
├── master-4, master-5
├── volume-dc2-1, volume-dc2-2, volume-dc2-3
└── filer-3, filer-4
```

**Конфигурация:**

```bash
# DC1 volumes
weed volume -ip=volume-dc1-1 -dataCenter=dc1 -rack=rack-1 ...
weed volume -ip=volume-dc1-2 -dataCenter=dc1 -rack=rack-2 ...

# DC2 volumes
weed volume -ip=volume-dc2-1 -dataCenter=dc2 -rack=rack-1 ...
weed volume -ip=volume-dc2-2 -dataCenter=dc2 -rack=rack-2 ...
```

**С репликацией `100` (cross-DC):**
- Основная копия: dc1
- Реплика: dc2
- При потере dc1 данные доступны из dc2

### Проверка репликации

```bash
# Проверка топологии
curl http://master-1:9333/dir/status?pretty=y

# Проверка распределения volumes
weed shell -master=master-1:9333 << EOF
volume.list
EOF

# Проверка integrity дисков
weed shell -master=master-1:9333 << EOF
volume.check.disk
EOF

# Проверка репликации конкретного файла
curl http://master-1:9333/dir/lookup?volumeId=3

# Ответ покажет все реплики:
# {
#   "volumeId": "3",
#   "locations": [
#     {"url": "volume-1-1:8080", "publicUrl": "volume-1-1:8080"},
#     {"url": "volume-2-1:8080", "publicUrl": "volume-2-1:8080"}
#   ]
# }
```

### Восстановление репликации

**Автоматическое восстановление:**

SeaweedFS автоматически восстанавливает потерянные реплики при выходе сервера из строя.

```bash
# Проверка потерянных реплик
weed shell -master=master-1:9333 << EOF
volume.check.disk -volumeId=3
EOF

# Автоматическое восстановление запускается мастером
# Логи на master сервере покажут процесс
```

**Ручное восстановление репликации:**

```bash
# Ребалансировка кластера
weed shell -master=master-1:9333 << EOF
volume.balance -force
EOF

# Принудительная репликация volume
weed shell -master=master-1:9333 << EOF
volume.fix.replication
EOF
```

### Сценарии отказов

**Сценарий 1: Потеря одного volume сервера**

```
Конфигурация: 9 volume серверов, репликация 010
Результат: Данные доступны из других rack
Действия: 
1. Система автоматически помечает сервер как недоступный
2. Запросы перенаправляются на реплики
3. После восстановления сервера данные синхронизируются
```

**Сценарий 2: Потеря целого rack**

```
Конфигурация: 3 rack, репликация 010
Результат: Данные доступны из других rack
Действия:
1. Master перенаправляет запросы на оставшиеся rack
2. Производительность может снизиться из-за повышенной нагрузки
3. После восстановления rack запускается репликация
```

**Сценарий 3: Потеря master leader**

```
Конфигурация: 3 master серверов
Результат: Автоматический failover через Raft
Действия:
1. Оставшиеся master серверы выбирают нового лидера (~5-10 секунд)
2. Клиенты автоматически переключаются на нового лидера
3. Восстановленный master становится follower
```

---

## Балансировка нагрузки

### Nginx для Master серверов

Создайте `/etc/nginx/conf.d/seaweedfs-master.conf`:

```nginx
upstream seaweedfs_masters {
    least_conn;
    
    server master-1:9333 max_fails=3 fail_timeout=30s;
    server master-2:9333 max_fails=3 fail_timeout=30s backup;
    server master-3:9333 max_fails=3 fail_timeout=30s backup;
    
    keepalive 64;
}

server {
    listen 9333;
    
    access_log /var/log/nginx/seaweedfs-master-access.log;
    error_log /var/log/nginx/seaweedfs-master-error.log;
    
    location / {
        proxy_pass http://seaweedfs_masters;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
    
    # Health check endpoint
    location /cluster/status {
        proxy_pass http://seaweedfs_masters;
        access_log off;
    }
}
```

### Nginx для Filer серверов

Создайте `/etc/nginx/conf.d/seaweedfs-filer.conf`:

```nginx
upstream seaweedfs_filers {
    least_conn;
    
    server filer-1:8888 max_fails=3 fail_timeout=30s;
    server filer-2:8888 max_fails=3 fail_timeout=30s;
    server filer-3:8888 max_fails=3 fail_timeout=30s;
    
    keepalive 128;
}

server {
    listen 8888;
    server_name filer.example.com;
    
    client_max_body_size 10G;
    client_body_buffer_size 128k;
    
    access_log /var/log/nginx/seaweedfs-filer-access.log;
    error_log /var/log/nginx/seaweedfs-filer-error.log;
    
    location / {
        proxy_pass http://seaweedfs_filers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Buffering для больших файлов
        proxy_buffering off;
        proxy_request_buffering off;
    }
}
```

### Nginx для S3 Gateway

Создайте `/etc/nginx/conf.d/seaweedfs-s3.conf`:

```nginx
upstream seaweedfs_s3 {
    least_conn;
    
    server filer-1:8333 max_fails=3 fail_timeout=30s;
    server filer-2:8333 max_fails=3 fail_timeout=30s;
    server filer-3:8333 max_fails=3 fail_timeout=30s;
    
    keepalive 128;
}

server {
    listen 80;
    listen 443 ssl http2;
    server_name s3.example.com *.s3.example.com;
    
    # SSL configuration
    ssl_certificate /etc/nginx/ssl/s3.example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/s3.example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    client_max_body_size 100G;
    client_body_buffer_size 128k;
    
    # Логирование
    access_log /var/log/nginx/seaweedfs-s3-access.log;
    error_log /var/log/nginx/seaweedfs-s3-error.log;
    
    # Основной location
    location / {
        proxy_pass http://seaweedfs_s3;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Отключение буферизации для больших файлов
        proxy_buffering off;
        proxy_request_buffering off;
    }
    
    # Health check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### HAProxy альтернатива

Создайте `/etc/haproxy/haproxy.cfg`:

```
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4096

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  300000
    timeout server  300000

# Master servers
frontend master_frontend
    bind *:9333
    default_backend master_backend

backend master_backend
    balance leastconn
    option httpchk GET /cluster/status
    http-check expect status 200
    
    server master-1 master-1:9333 check inter 5s fall 3 rise 2
    server master-2 master-2:9333 check inter 5s fall 3 rise 2 backup
    server master-3 master-3:9333 check inter 5s fall 3 rise 2 backup

# Filer servers
frontend filer_frontend
    bind *:8888
    default_backend filer_backend

backend filer_backend
    balance leastconn
    option httpchk GET /
    http-check expect status 200
    
    server filer-1 filer-1:8888 check inter 5s fall 3 rise 2
    server filer-2 filer-2:8888 check inter 5s fall 3 rise 2
    server filer-3 filer-3:8888 check inter 5s fall 3 rise 2

# S3 Gateway
frontend s3_frontend
    bind *:8333
    default_backend s3_backend

backend s3_backend
    balance leastconn
    option httpchk GET /
    
    server filer-1 filer-1:8333 check inter 5s fall 3 rise 2
    server filer-2 filer-2:8333 check inter 5s fall 3 rise 2
    server filer-3 filer-3:8333 check inter 5s fall 3 rise 2

# Statistics
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
```

---

## Мониторинг и алертинг

### Prometheus конфигурация
