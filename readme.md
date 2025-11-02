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

Создайте `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'seaweedfs-production'
    environment: 'production'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Rule files
rule_files:
  - '/etc/prometheus/rules/seaweedfs_alerts.yml'

# Scrape configurations
scrape_configs:
  # Master servers
  - job_name: 'seaweedfs-master'
    static_configs:
      - targets:
          - 'master-1:19333'
          - 'master-2:19333'
          - 'master-3:19333'
        labels:
          component: 'master'
          datacenter: 'dc1'

  # Volume servers
  - job_name: 'seaweedfs-volume'
    static_configs:
      - targets:
          - 'volume-1-1:18080'
          - 'volume-1-2:18080'
          - 'volume-1-3:18080'
        labels:
          component: 'volume'
          rack: 'rack-1'
          datacenter: 'dc1'
      - targets:
          - 'volume-2-1:18080'
          - 'volume-2-2:18080'
          - 'volume-2-3:18080'
        labels:
          component: 'volume'
          rack: 'rack-2'
          datacenter: 'dc1'
      - targets:
          - 'volume-3-1:18080'
          - 'volume-3-2:18080'
          - 'volume-3-3:18080'
        labels:
          component: 'volume'
          rack: 'rack-3'
          datacenter: 'dc1'

  # Filer servers
  - job_name: 'seaweedfs-filer'
    static_configs:
      - targets:
          - 'filer-1:18888'
          - 'filer-2:18888'
          - 'filer-3:18888'
        labels:
          component: 'filer'
          datacenter: 'dc1'

  # Node exporters
  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'master-1:9100'
          - 'master-2:9100'
          - 'master-3:9100'
          - 'volume-1-1:9100'
          - 'volume-2-1:9100'
          - 'volume-3-1:9100'
          - 'filer-1:9100'
          - 'filer-2:9100'
          - 'filer-3:9100'
```

### Alert Rules для SeaweedFS

Создайте `/etc/prometheus/rules/seaweedfs_alerts.yml`:

```yaml
groups:
  - name: seaweedfs_master
    interval: 30s
    rules:
      # Master не доступен
      - alert: MasterServerDown
        expr: up{job="seaweedfs-master"} == 0
        for: 2m
        labels:
          severity: critical
          component: master
        annotations:
          summary: "Master server {{ $labels.instance }} is down"
          description: "Master server {{ $labels.instance }} has been down for more than 2 minutes."

      # Нет лидера в кластере
      - alert: MasterNoLeader
        expr: sum(weed_master_is_leader) == 0
        for: 1m
        labels:
          severity: critical
          component: master
        annotations:
          summary: "No master leader elected"
          description: "SeaweedFS cluster has no master leader for more than 1 minute."

      # Высокая задержка на master
      - alert: MasterHighLatency
        expr: histogram_quantile(0.99, rate(weed_master_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
          component: master
        annotations:
          summary: "High latency on master {{ $labels.instance }}"
          description: "99th percentile latency is {{ $value }}s on {{ $labels.instance }}"

  - name: seaweedfs_volume
    interval: 30s
    rules:
      # Volume сервер не доступен
      - alert: VolumeServerDown
        expr: up{job="seaweedfs-volume"} == 0
        for: 3m
        labels:
          severity: warning
          component: volume
        annotations:
          summary: "Volume server {{ $labels.instance }} is down"
          description: "Volume server {{ $labels.instance }} in rack {{ $labels.rack }} has been down for more than 3 minutes."

      # Недостаточно свободного места
      - alert: VolumeServerDiskSpaceLow
        expr: (weed_volume_total_disk_size - weed_volume_used_disk_size) / weed_volume_total_disk_size * 100 < 10
        for: 5m
        labels:
          severity: warning
          component: volume
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Volume server {{ $labels.instance }} has less than 10% free disk space ({{ $value }}% remaining)."

      # Критически мало свободного места
      - alert: VolumeServerDiskSpaceCritical
        expr: (weed_volume_total_disk_size - weed_volume_used_disk_size) / weed_volume_total_disk_size * 100 < 5
        for: 2m
        labels:
          severity: critical
          component: volume
        annotations:
          summary: "Critical disk space on {{ $labels.instance }}"
          description: "Volume server {{ $labels.instance }} has less than 5% free disk space ({{ $value }}% remaining). Immediate action required!"

      # Высокое количество ошибок чтения
      - alert: VolumeHighReadErrors
        expr: rate(weed_volume_read_errors_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
          component: volume
        annotations:
          summary: "High read error rate on {{ $labels.instance }}"
          description: "Volume server {{ $labels.instance }} has {{ $value }} read errors per second."

      # Высокое количество ошибок записи
      - alert: VolumeHighWriteErrors
        expr: rate(weed_volume_write_errors_total[5m]) > 5
        for: 5m
        labels:
          severity: critical
          component: volume
        annotations:
          summary: "High write error rate on {{ $labels.instance }}"
          description: "Volume server {{ $labels.instance }} has {{ $value }} write errors per second."

      # Недостаточно volumes
      - alert: LowAvailableVolumes
        expr: weed_master_volume_free_count < 10
        for: 5m
        labels:
          severity: warning
          component: volume
        annotations:
          summary: "Low number of available volumes"
          description: "Only {{ $value }} free volumes available in the cluster. Consider adding more volume capacity."

  - name: seaweedfs_filer
    interval: 30s
    rules:
      # Filer не доступен
      - alert: FilerServerDown
        expr: up{job="seaweedfs-filer"} == 0
        for: 2m
        labels:
          severity: critical
          component: filer
        annotations:
          summary: "Filer server {{ $labels.instance }} is down"
          description: "Filer server {{ $labels.instance }} has been down for more than 2 minutes."

      # Высокая задержка на filer
      - alert: FilerHighLatency
        expr: histogram_quantile(0.99, rate(weed_filer_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
          component: filer
        annotations:
          summary: "High latency on filer {{ $labels.instance }}"
          description: "99th percentile latency is {{ $value }}s on {{ $labels.instance }}"

      # Проблемы с базой данных метаданных
      - alert: FilerDatabaseErrors
        expr: rate(weed_filer_store_errors_total[5m]) > 1
        for: 5m
        labels:
          severity: critical
          component: filer
        annotations:
          summary: "Database errors on filer {{ $labels.instance }}"
          description: "Filer {{ $labels.instance }} experiencing {{ $value }} database errors per second."

  - name: seaweedfs_replication
    interval: 60s
    rules:
      # Проблемы с репликацией
      - alert: ReplicationLag
        expr: weed_volume_replication_lag_seconds > 300
        for: 10m
        labels:
          severity: warning
          component: replication
        annotations:
          summary: "Replication lag detected"
          description: "Replication lag is {{ $value }}s on volume {{ $labels.volume_id }}. Check volume server health."

      # Потеря реплик
      - alert: MissingReplicas
        expr: weed_volume_replica_count < weed_volume_expected_replica_count
        for: 5m
        labels:
          severity: critical
          component: replication
        annotations:
          summary: "Missing replicas for volume {{ $labels.volume_id }}"
          description: "Volume {{ $labels.volume_id }} has {{ $value }} replicas instead of expected {{ $labels.expected }}."
```

### Grafana Dashboard

Создайте dashboard с следующими панелями:

**Импортируйте готовый dashboard:**

```bash
# Скачайте dashboard JSON
curl -o seaweedfs-dashboard.json https://raw.githubusercontent.com/seaweedfs/seaweedfs/master/docker/grafana-dashboard.json

# Импортируйте через Grafana UI или API
curl -X POST http://admin:admin@grafana:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @seaweedfs-dashboard.json
```

**Основные метрики для мониторинга:**

| Метрика | Описание | Threshold |
|---------|----------|-----------|
| `up{job="seaweedfs-master"}` | Доступность master серверов | = 1 |
| `weed_master_is_leader` | Наличие лидера | sum() > 0 |
| `weed_master_volume_free_count` | Количество свободных volumes | > 10 |
| `weed_volume_total_disk_size` | Общий объем дисков | - |
| `weed_volume_used_disk_size` | Использованный объем | < 90% |
| `rate(weed_volume_request_total[5m])` | RPS на volume серверах | - |
| `weed_volume_replica_count` | Количество реплик | = expected |
| `rate(weed_filer_request_total[5m])` | RPS на filer серверах | - |

### Node Exporter для системных метрик

**Установка на каждом сервере:**

```bash
# Скачивание
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter-*.tar.gz
sudo cp node_exporter-*/node_exporter /usr/local/bin/
sudo chown root:root /usr/local/bin/node_exporter

# Создание systemd service
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.filesystem.mount-points-exclude='^/(sys|proc|dev|host|etc)($$|/)' \
  --collector.diskstats.ignored-devices='^(ram|loop|fd|(h|s|v|xv)d[a-z]|nvme\\d+n\\d+p)\\d+$$'

[Install]
WantedBy=multi-user.target
EOF

# Создание пользователя
sudo useradd -r -s /bin/false node_exporter

# Запуск
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

### Alertmanager конфигурация

Создайте `/etc/alertmanager/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  
  routes:
    # Critical alerts - немедленная отправка
    - match:
        severity: critical
      receiver: 'critical-alerts'
      group_wait: 0s
      repeat_interval: 4h
    
    # Warning alerts - группировка
    - match:
        severity: warning
      receiver: 'warning-alerts'
      group_wait: 30s
      repeat_interval: 12h

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'
        send_resolved: true

  - name: 'critical-alerts'
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#critical-alerts'
        title: 'SeaweedFS Critical Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'

  - name: 'warning-alerts'
    email_configs:
      - to: 'team@example.com'
        send_resolved: true
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#seaweedfs-alerts'
        title: 'SeaweedFS Warning'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'

inhibit_rules:
  # Не отправлять warning если есть critical
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### Логирование

**Централизованное логирование с Loki:**

```bash
# Установка Promtail на каждом сервере
wget https://github.com/grafana/loki/releases/download/v2.9.0/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
```

**Конфигурация Promtail (`/etc/promtail/config.yml`):**

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/log/promtail/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: seaweedfs
    static_configs:
      - targets:
          - localhost
        labels:
          job: seaweedfs
          host: ${HOSTNAME}
          __path__: /var/log/seaweedfs/*.log
    
    pipeline_stages:
      - match:
          selector: '{job="seaweedfs"}'
          stages:
            - regex:
                expression: '^(?P<timestamp>\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2}) \[(?P<level>\w+)\] (?P<message>.*)$'
            - labels:
                level:
            - timestamp:
                source: timestamp
                format: '2006/01/02 15:04:05'

  - job_name: journal
    journal:
      max_age: 12h
      labels:
        job: systemd-journal
        host: ${HOSTNAME}
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
      - source_labels: ['__journal__hostname']
        target_label: 'host'
```

### Health Check скрипты

**Полная проверка здоровья кластера (`/usr/local/bin/check-cluster-health.sh`):**

```bash
#!/bin/bash

set -euo pipefail

MASTER_HOST="${MASTER_HOST:-master-1:9333}"
ERRORS=0

echo "=== SeaweedFS Cluster Health Check ==="
echo "Date: $(date)"
echo ""

# Проверка Master серверов
echo "Checking Master servers..."
LEADER=$(curl -sf "http://${MASTER_HOST}/cluster/status" | jq -r '.Leader' || echo "")
if [ -z "$LEADER" ]; then
    echo "❌ ERROR: No master leader found"
    ERRORS=$((ERRORS + 1))
else
    echo "✓ Master leader: $LEADER"
fi

PEERS=$(curl -sf "http://${MASTER_HOST}/cluster/status" | jq -r '.Peers[]' || echo "")
PEER_COUNT=$(echo "$PEERS" | wc -l)
if [ "$PEER_COUNT" -lt 3 ]; then
    echo "❌ ERROR: Only $PEER_COUNT master peers found (expected 3+)"
    ERRORS=$((ERRORS + 1))
else
    echo "✓ Master peers: $PEER_COUNT"
fi

# Проверка Volume серверов
echo ""
echo "Checking Volume servers..."
TOPOLOGY=$(curl -sf "http://${MASTER_HOST}/dir/status" | jq -r '.Topology')
VOLUME_COUNT=$(echo "$TOPOLOGY" | jq -r '.DataCenters[].Racks[].DataNodes[] | .Volumes' | wc -l)
if [ "$VOLUME_COUNT" -lt 1 ]; then
    echo "❌ ERROR: No volume servers found"
    ERRORS=$((ERRORS + 1))
else
    echo "✓ Volume servers connected: $VOLUME_COUNT"
fi

FREE_VOLUMES=$(curl -sf "http://${MASTER_HOST}/dir/status" | jq -r '.Topology.Free')
if [ "$FREE_VOLUMES" -lt 10 ]; then
    echo "⚠️  WARNING: Low free volumes: $FREE_VOLUMES (threshold: 10)"
else
    echo "✓ Free volumes: $FREE_VOLUMES"
fi

# Проверка репликации
echo ""
echo "Checking replication..."
MISSING_REPLICAS=$(curl -sf "http://${MASTER_HOST}/vol/status?collection=*" | \
    jq -r '.Volumes[] | select(.ReplicaPlacement != .FileCount) | .Id' | wc -l)
if [ "$MISSING_REPLICAS" -gt 0 ]; then
    echo "❌ ERROR: $MISSING_REPLICAS volumes with replication issues"
    ERRORS=$((ERRORS + 1))
else
    echo "✓ All volumes properly replicated"
fi

# Проверка дискового пространства
echo ""
echo "Checking disk space..."
USED_PERCENT=$(curl -sf "http://${MASTER_HOST}/dir/status" | \
    jq -r '(.Topology.Max - .Topology.Free) / .Topology.Max * 100')
if (( $(echo "$USED_PERCENT > 90" | bc -l) )); then
    echo "❌ ERROR: Disk usage critical: ${USED_PERCENT}%"
    ERRORS=$((ERRORS + 1))
elif (( $(echo "$USED_PERCENT > 80" | bc -l) )); then
    echo "⚠️  WARNING: Disk usage high: ${USED_PERCENT}%"
else
    echo "✓ Disk usage: ${USED_PERCENT}%"
fi

# Результат
echo ""
echo "======================================="
if [ "$ERRORS" -eq 0 ]; then
    echo "✓ Cluster health check PASSED"
    exit 0
else
    echo "❌ Cluster health check FAILED with $ERRORS errors"
    exit 1
fi
```

```bash
chmod +x /usr/local/bin/check-cluster-health.sh

# Добавить в cron для периодической проверки
echo "*/5 * * * * /usr/local/bin/check-cluster-health.sh >> /var/log/seaweedfs/health-check.log 2>&1" | sudo crontab -
```

---

## Backup и восстановление

### Стратегии резервного копирования

**Три уровня backup:**

1. **Метаданные Master серверов** — критичны для топологии кластера
2. **Метаданные Filer (база данных)** — критичны для файловой структуры
3. **Volume данные** — сами файлы (уже защищены репликацией)

### Backup Master метаданных

**Автоматический backup скрипт (`/usr/local/bin/backup-master.sh`):**

```bash
#!/bin/bash

set -euo pipefail

BACKUP_DIR="/backup/seaweedfs/master"
MASTER_DATA_DIR="/data/seaweedfs/metadata"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Создание директории backup
mkdir -p "$BACKUP_DIR"

# Backup метаданных
echo "Creating master metadata backup..."
tar -czf "$BACKUP_DIR/master-metadata-$DATE.tar.gz" \
    -C "$(dirname $MASTER_DATA_DIR)" \
    "$(basename $MASTER_DATA_DIR)"

# Проверка backup
if [ -f "$BACKUP_DIR/master-metadata-$DATE.tar.gz" ]; then
    echo "✓ Backup created: master-metadata-$DATE.tar.gz"
    
    # Размер backup
    SIZE=$(du -h "$BACKUP_DIR/master-metadata-$DATE.tar.gz" | cut -f1)
    echo "Backup size: $SIZE"
else
    echo "❌ ERROR: Backup failed"
    exit 1
fi

# Удаление старых backup
echo "Cleaning old backups (older than $RETENTION_DAYS days)..."
find "$BACKUP_DIR" -name "master-metadata-*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Список backup
echo ""
echo "Available backups:"
ls -lh "$BACKUP_DIR"/master-metadata-*.tar.gz | tail -10

# Отправка на удаленное хранилище (опционально)
if [ -n "${REMOTE_BACKUP:-}" ]; then
    echo "Uploading to remote storage..."
    aws s3 cp "$BACKUP_DIR/master-metadata-$DATE.tar.gz" \
        "s3://your-backup-bucket/seaweedfs/master/" \
        --endpoint-url "$REMOTE_BACKUP"
fi
```

**Настройка периодического backup:**

```bash
chmod +x /usr/local/bin/backup-master.sh

# Добавить в crontab (ежедневно в 2 AM)
echo "0 2 * * * /usr/local/bin/backup-master.sh >> /var/log/seaweedfs/backup-master.log 2>&1" | sudo crontab -
```

### Backup Filer метаданных (PostgreSQL)

**Backup скрипт для PostgreSQL (`/usr/local/bin/backup-filer-db.sh`):**

```bash
#!/bin/bash

set -euo pipefail

BACKUP_DIR="/backup/seaweedfs/filer"
DB_HOST="postgres-host"
DB_NAME="seaweedfs"
DB_USER="seaweedfs"
DB_PASSWORD="secure_password_here"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Создание директории
mkdir -p "$BACKUP_DIR"

# Backup базы данных
echo "Creating filer database backup..."
PGPASSWORD="$DB_PASSWORD" pg_dump \
    -h "$DB_HOST" \
    -U "$DB_USER" \
    -d "$DB_NAME" \
    -F c \
    -f "$BACKUP_DIR/filer-db-$DATE.dump"

# Проверка backup
if [ -f "$BACKUP_DIR/filer-db-$DATE.dump" ]; then
    echo "✓ Database backup created: filer-db-$DATE.dump"
    
    SIZE=$(du -h "$BACKUP_DIR/filer-db-$DATE.dump" | cut -f1)
    echo "Backup size: $SIZE"
    
    # Сжатие
    gzip "$BACKUP_DIR/filer-db-$DATE.dump"
    echo "✓ Backup compressed: filer-db-$DATE.dump.gz"
else
    echo "❌ ERROR: Database backup failed"
    exit 1
fi

# Удаление старых backup
echo "Cleaning old backups..."
find "$BACKUP_DIR" -name "filer-db-*.dump.gz" -mtime +$RETENTION_DAYS -delete

# Список backup
echo ""
echo "Available backups:"
ls -lh "$BACKUP_DIR"/filer-db-*.dump.gz | tail -10

# Отправка на удаленное хранилище
if [ -n "${REMOTE_BACKUP:-}" ]; then
    echo "Uploading to remote storage..."
    aws s3 cp "$BACKUP_DIR/filer-db-$DATE.dump.gz" \
        "s3://your-backup-bucket/seaweedfs/filer/" \
        --endpoint-url "$REMOTE_BACKUP"
fi
```

```bash
chmod +x /usr/local/bin/backup-filer-db.sh

# Ежедневный backup в 3 AM
echo "0 3 * * * /usr/local/bin/backup-filer-db.sh >> /var/log/seaweedfs/backup-filer.log 2>&1" | sudo crontab -
```

### Backup Volume данных

**Подходы к backup volumes:**

**1. Snapshot-based backup (рекомендуется для production):**

```bash
#!/bin/bash
# Создание snapshot для каждого volume диска

VOLUME_DIRS=(
    "/data/disk1/seaweedfs/volumes"
    "/data/disk2/seaweedfs/volumes"
    "/data/disk3/seaweedfs/volumes"
    "/data/disk4/seaweedfs/volumes"
)

for DIR in "${VOLUME_DIRS[@]}"; do
    DISK=$(df "$DIR" | tail -1 | awk '{print $1}')
    SNAPSHOT_NAME="snapshot-$(date +%Y%m%d_%H%M%S)"
    
    echo "Creating snapshot for $DISK..."
    # Для LVM
    lvcreate -L 10G -s -n "$SNAPSHOT_NAME" "$DISK"
    
    # Монтирование snapshot
    mkdir -p "/mnt/snapshots/$SNAPSHOT_NAME"
    mount "/dev/vg0/$SNAPSHOT_NAME" "/mnt/snapshots/$SNAPSHOT_NAME"
    
    # Backup snapshot
    rsync -a "/mnt/snapshots/$SNAPSHOT_NAME/" "/backup/volumes/$SNAPSHOT_NAME/"
    
    # Очистка
    umount "/mnt/snapshots/$SNAPSHOT_NAME"
    lvremove -f "/dev/vg0/$SNAPSHOT_NAME"
done
```

**2. Incremental backup с rsync:**

```bash
#!/bin/bash
# Инкрементальный backup volumes

BACKUP_BASE="/backup/seaweedfs/volumes"
VOLUME_DIRS="/data/disk*/seaweedfs/volumes"
DATE=$(date +%Y%m%d)

for SOURCE in $VOLUME_DIRS; do
    DISK_NAME=$(basename $(dirname $(dirname "$SOURCE")))
    DEST="$BACKUP_BASE/$DISK_NAME"
    
    echo "Backing up $SOURCE to $DEST..."
    
    rsync -a --delete \
        --link-dest="$DEST/current" \
        "$SOURCE/" \
        "$DEST/$DATE/"
    
    # Обновление симлинка на последний backup
    rm -f "$DEST/current"
    ln -s "$DATE" "$DEST/current"
done
```

**3. Репликация в другой датацентр (best practice):**

```bash
# Настройка cross-DC репликации уже защищает данные
# Используйте репликацию 100 (cross-DC) для critical данных
weed shell -master=master-1:9333 << EOF
volume.create -replication=100 -collection=critical-data
EOF
```

### Восстановление Master сервера

**Восстановление из backup:**

```bash
#!/bin/bash
# restore-master.sh

BACKUP_FILE="$1"
MASTER_DATA_DIR="/data/seaweedfs/metadata"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.tar.gz>"
    exit 1
fi

# Остановка сервиса
echo "Stopping master service..."
sudo systemctl stop seaweedfs-master

# Backup текущих данных (на всякий случай)
echo "Creating safety backup..."
sudo mv "$MASTER_DATA_DIR" "${MASTER_DATA_DIR}.backup-$(date +%Y%m%d_%H%M%S)"

# Восстановление из backup
echo "Restoring from $BACKUP_FILE..."
sudo mkdir -p "$(dirname $MASTER_DATA_DIR)"
sudo tar -xzf "$BACKUP_FILE" -C "$(dirname $MASTER_DATA_DIR)"

# Проверка прав доступа
sudo chown -R seaweedfs:seaweedfs "$MASTER_DATA_DIR"
sudo chmod 750 "$MASTER_DATA_DIR"

# Запуск сервиса
echo "Starting master service..."
sudo systemctl start seaweedfs-master

# Проверка статуса
sleep 5
if sudo systemctl is-active --quiet seaweedfs-master; then
    echo "✓ Master service restored successfully"
    
    # Проверка кластера
    curl -sf "http://localhost:9333/cluster/status?pretty=y"
else
    echo "❌ ERROR: Master service failed to start"
    echo "Checking logs..."
    sudo journalctl -u seaweedfs-master -n 50
    exit 1
fi
```

### Восстановление Filer метаданных

**Восстановление PostgreSQL базы данных:**

```bash
#!/bin/bash
# restore-filer-db.sh

BACKUP_FILE="$1"
DB_HOST="postgres-host"
DB_NAME="seaweedfs"
DB_USER="seaweedfs"
DB_PASSWORD="secure_password_here"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file.dump.gz>"
    exit 1
fi

# Остановка filer сервисов
echo "Stopping filer services..."
sudo systemctl stop seaweedfs-filer

# Распаковка backup
echo "Decompressing backup..."
gunzip -c "$BACKUP_FILE" > /tmp/filer-restore.dump

# Создание новой базы данных
echo "Recreating database..."
PGPASSWORD="$DB_PASSWORD" psql -h "$DB_HOST" -U postgres << EOF
DROP DATABASE IF EXISTS ${DB_NAME}_old;
ALTER DATABASE $DB_NAME RENAME TO ${DB_NAME}_old;
CREATE DATABASE $DB_NAME;
GRANT ALL PRIVILEGES ON DATABASE $DB_NAME TO $DB_USER;
EOF

# Восстановление данных
echo "Restoring database..."
PGPASSWORD="$DB_PASSWORD" pg_restore \
    -h "$DB_HOST" \
    -U "$DB_USER" \
    -d "$DB_NAME" \
    -v \
    /tmp/filer-restore.dump

# Проверка восстановления
RECORD_COUNT=$(PGPASSWORD="$DB_PASSWORD" psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -t -c "SELECT COUNT(*) FROM filemeta;")
echo "Records restored: $RECORD_COUNT"

# Очистка
rm /tmp/filer-restore.dump

# Запуск filer сервисов
echo "Starting filer services..."
sudo systemctl start seaweedfs-filer

# Проверка
sleep 5
if sudo systemctl is-active --quiet seaweedfs-filer; then
    echo "✓ Filer database restored successfully"
else
    echo "❌ ERROR: Filer service failed to start"
    sudo journalctl -u seaweedfs-filer -n 50
    exit 1
fi
```

### Disaster Recovery Plan

**Сценарий 1: Полная потеря одного master сервера**

```bash
# 1. Удалить мертвый сервер из peers на оставшихся master
weed shell -master=master-2:9333 << EOF
cluster.raft.remove -id=master-1:9333
EOF

# 2. Поднять новый master сервер
# 3. Добавить в кластер
weed shell -master=master-2:9333 << EOF
cluster.raft.add -id=master-1-new:9333
EOF
```

**Сценарий 2: Потеря всего master кластера**

```bash
# 1. Восстановить последний backup на 3 новых серверах
for HOST in master-1 master-2 master-3; do
    scp /backup/master-metadata-latest.tar.gz $HOST:/tmp/
    ssh $HOST "sudo /usr/local/bin/restore-master.sh /tmp/master-metadata-latest.tar.gz"
done

# 2. Запустить master кластер с правильными peers
# 3. Проверить topology
curl http://master-1:9333/dir/status?pretty=y
```

**Сценарий 3: Потеря базы данных Filer**

```bash
# 1. Восстановить последний backup PostgreSQL
/usr/local/bin/restore-filer-db.sh /backup/filer-db-latest.dump.gz

# 2. Перезапустить все filer серверы
for HOST in filer-1 filer-2 filer-3; do
    ssh $HOST "sudo systemctl restart seaweedfs-filer"
done

# 3. Проверить доступность
curl http://filer-1:8888/?pretty=y
```

### Тестирование backup

**Регулярное тестирование восстановления (ежемесячно):**

```bash
#!/bin/bash
# test-restore.sh

TEST_DATE=$(date +%Y%m%d)
TEST_DIR="/tmp/restore-test-$TEST_DATE"

echo "=== Testing Backup Restore ==="
echo "Date: $(date)"

# 1. Тест восстановления master
echo ""
echo "Testing master backup restore..."
mkdir -p "$TEST_DIR/master"
tar -xzf /backup/seaweedfs/master/master-metadata-latest.tar.gz -C "$TEST_DIR/master"
if [ $? -eq 0 ]; then
    echo "✓ Master backup is valid"
else
    echo "❌ Master backup is corrupted!"
    exit 1
fi

# 2. Тест восстановления filer DB
echo ""
echo "Testing filer database backup restore..."
gunzip -c /backup/seaweedfs/filer/filer-db-latest.dump.gz > "$TEST_DIR/filer-test.dump"
if [ $? -eq 0 ]; then
    echo "✓ Filer backup is valid"
else
    echo "❌ Filer backup is corrupted!"
    exit 1
fi

# 3. Проверка целостности
echo ""
echo "Checking backup integrity..."
MASTER_SIZE=$(du -sh /backup/seaweedfs/master/master-metadata-latest.tar.gz | cut -f1)
FILER_SIZE=$(du -sh /backup/seaweedfs/filer/filer-db-latest.dump.gz | cut -f1)

echo "Master backup size: $MASTER_SIZE"
echo "Filer backup size: $FILER_SIZE"

# Очистка
rm -rf "$TEST_DIR"

echo ""
echo "✓ All backup tests passed!"
```

```bash
chmod +x /usr/local/bin/test-restore.sh

# Ежемесячное тестирование
echo "0 4 1 * * /usr/local/bin/test-restore.sh >> /var/log/seaweedfs/restore-test.log 2>&1" | sudo crontab -
```

---

## Оптимизация производительности

### Настройка производительности Volume серверов

**Оптимизация файловой системы:**

```bash
# XFS mount options для максимальной производительности
# В /etc/fstab:
UUID=xxx /data/disk1 xfs noatime,nodiratime,logbufs=8,logbsize=256k,largeio,inode64,swalloc 0 2

# Remount с новыми параметрами
sudo mount -o remount /data/disk1
```

**Настройка I/O scheduler:**

```bash
# Для SSD используйте none или mq-deadline
echo "none" | sudo tee /sys/block/sdb/queue/scheduler

# Для HDD используйте mq-deadline
echo "mq-deadline" | sudo tee /sys/block/sdc/queue/scheduler

# Постоянная настройка через udev rule
cat <<EOF | sudo tee /etc/udev/rules.d/60-schedulers.rules
# SSD
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
# HDD
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
EOF
```

**Kernel parameters для высокой нагрузки:**

```bash
# Добавить в /etc/sysctl.conf
cat <<EOF | sudo tee -a /etc/sysctl.conf

# Network tuning
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.ip_local_port_range = 1024 65535

# File system tuning
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.aio-max-nr = 1048576

# Memory management
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.vfs_cache_pressure = 50

# Increase TCP buffer sizes
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
EOF

# Применить настройки
sudo sysctl -p
```

**Limits для seaweedfs пользователя:**

```bash
# В /etc/security/limits.conf
cat <<EOF | sudo tee -a /etc/security/limits.conf
seaweedfs soft nofile 65536
seaweedfs hard nofile 65536
seaweedfs soft nproc 32768
seaweedfs hard nproc 32768
EOF
```

### Оптимизация Volume конфигурации

**Подбор оптимального количества volumes:**

```
Общая формула:
volumes_per_disk = disk_size_gb / volume_size_mb

Пример для 4TB диска:
4000 GB / 30 GB = ~133 volumes
Рекомендуется: 100-120 volumes (с запасом)

Для параметра -max:
-max=25 означает максимум 25 volumes на одну -dir
Если у вас 4 диска: 4 × 25 = 100 volumes total
```

**Оптимальные параметры Volume:**

```ini
# В systemd service
ExecStart=/usr/local/bin/weed volume \
  # Размер volume файла (рекомендуется 30GB)
  -volumeSizeLimitMB=30000 \
  
  # Максимум volumes на директорию
  -max=25 \
  
  # Тип индекса (leveldb быстрее)
  -index=leveldb \
  
  # Ограничение скорости компрессии (MB/s)
  -compactionMBps=100 \
  
  # Минимальный процент свободного места
  -minFreeSpacePercent=5 \
  
  # Чтение напрямую с диска (для больших файлов)
  -readMode=os \
  
  # Количество потоков для компрессии
  -parallelUpload=8 \
  
  # Timeout для репликации
  -replicationTimeout=90s
```

### Оптимизация Filer производительности

**Настройка PostgreSQL для Filer:**

```sql
-- В postgresql.conf

-- Memory settings (для 16GB RAM сервера)
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
work_mem = 64MB

-- Checkpoint settings
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100

-- Query planner
random_page_cost = 1.1  -- Для SSD
effective_io_concurrency = 200

-- Write performance
synchronous_commit = off  -- Осторожно: может привести к потере данных
wal_writer_delay = 200ms
commit_delay = 100

-- Autovacuum tuning
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 2ms

-- Connections
max_connections = 200
```

**Индексы для оптимизации запросов:**

```sql
-- Дополнительные индексы для быстрого поиска
CREATE INDEX CONCURRENTLY idx_filemeta_directory_name ON filemeta(directory, name);
CREATE INDEX CONCURRENTLY idx_filemeta_meta_gin ON filemeta USING gin(meta jsonb_path_ops);

-- Анализ таблицы
ANALYZE filemeta;

-- Проверка использования индексов
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
```

**Connection pooling с PgBouncer:**

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
seaweedfs = host=postgres-host port=5432 dbname=seaweedfs

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool settings
pool_mode = transaction
max_client_conn = 500
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3

# Timeouts
server_idle_timeout = 600
server_connect_timeout = 15
```

### Кэширование

**Настройка Redis для кэширования метаданных:**

```toml
# В filer.toml добавить:

[redis_cluster2]
enabled = true
addresses = [
  "redis-1:6379",
  "redis-2:6379",
  "redis-3:6379"
]
password = ""
readOnly = false

# Кэширование на 1 час
[filer.options]
cache_capacity_mb = 1024
cache_to_filer_timeout_ms = 100
```

**Varnish для кэширования S3 запросов:**

```vcl
# /etc/varnish/default.vcl

vcl 4.0;

backend default {
    .host = "filer-lb";
    .port = "8333";
    .connect_timeout = 5s;
    .first_byte_timeout = 60s;
    .between_bytes_timeout = 10s;
}

sub vcl_recv {
    # Кэшировать GET и HEAD запросы
    if (req.method != "GET" && req.method != "HEAD") {
        return (pass);
    }
    
    # Не кэшировать bucket operations
    if (req.url ~ "^/\?") {
        return (pass);
    }
    
    return (hash);
}

sub vcl_backend_response {
    # Кэшировать на 1 час
    if (bereq.url ~ "\.(jpg|jpeg|png|gif|ico|svg|mp4|pdf|zip)$") {
        set beresp.ttl = 1h;
        set beresp.grace = 6h;
    }
    
    # Не кэшировать errors
    if (beresp.status >= 400) {
        set beresp.ttl = 0s;
    }
}

sub vcl_deliver {
    # Добавить заголовок с информацией о кэше
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }
}
```

### Benchmark и тестирование производительности

**Benchmark script для загрузки файлов:**

```bash
#!/bin/bash
# benchmark-upload.sh

FILER_HOST="filer-1:8888"
NUM_FILES=10000
FILE_SIZE_KB=100
THREADS=50

echo "=== SeaweedFS Upload Benchmark ==="
echo "Files: $NUM_FILES"
echo "Size: ${FILE_SIZE_KB}KB"
echo "Threads: $THREADS"
echo ""

# Создание тестового файла
dd if=/dev/urandom of=/tmp/test-file.bin bs=1024 count=$FILE_SIZE_KB 2>/dev/null

START_TIME=$(date +%s)

# Параллельная загрузка
seq 1 $NUM_FILES | xargs -P $THREADS -I {} bash -c "
    curl -sf -F 'file=@/tmp/test-file.bin' \
    'http://$FILER_HOST/benchmark/file-{}' > /dev/null
"

END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

# Расчет метрик
FILES_PER_SEC=$((NUM_FILES / DURATION))
MB_PER_SEC=$(((NUM_FILES * FILE_SIZE_KB / 1024) / DURATION))

echo "Duration: ${DURATION}s"
echo "Files/sec: $FILES_PER_SEC"
echo "MB/sec: $MB_PER_SEC"

# Очистка
rm /tmp/test-file.bin
```

**Benchmark для чтения:**

```bash
#!/bin/bash
# benchmark-download.sh

FILER_HOST="filer-1:8888"
NUM_REQUESTS=10000
THREADS=100

echo "=== SeaweedFS Download Benchmark ==="

START_TIME=$(date +%s)

seq 1 $NUM_REQUESTS | xargs -P $THREADS -I {} bash -c "
    FILE_NUM=\$((\$RANDOM % $NUM_FILES + 1))
    curl -sf 'http://$FILER_HOST/benchmark/file-\$FILE_NUM' > /dev/null
"

END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

REQUESTS_PER_SEC=$((NUM_REQUESTS / DURATION))

echo "Duration: ${DURATION}s"
echo "Requests/sec: $REQUESTS_PER_SEC"
```

**Load testing с wrk:**

```bash
# Установка wrk
git clone https://github.com/wg/wrk.git
cd wrk
make
sudo cp wrk /usr/local/bin/

# S3 API load test
wrk -t 12 -c 400 -d 60s --latency \
    -H "Authorization: AWS access_key:secret_key" \
    http://filer-1:8333/test-bucket/

# Filer API load test
wrk -t 12 -c 400 -d 60s --latency \
    http://filer-1:8888/test/
```

### Performance tuning чеклист

- [ ] XFS с оптимальными mount options
- [ ] I/O scheduler настроен (none для SSD, mq-deadline для HDD)
- [ ] Kernel parameters оптимизированы (sysctl.conf)
- [ ] File descriptors увеличены (ulimit)
- [ ] Volume размер оптимизирован (30GB рекомендуется)
- [ ] LevelDB выбран для индексов
- [ ] PostgreSQL настроен для высокой нагрузки
- [ ] Connection pooling настроен (PgBouncer)
- [ ] Redis кэширование включено
- [ ] CDN/Varnish для статики (опционально)
- [ ] Мониторинг производительности активен
- [ ] Регулярные benchmark тесты

---

## Troubleshooting

### Частые проблемы и решения

#### Проблема 1: Master не может выбрать лидера

**Симптомы:**
```
Error: no leader found
Raft cluster cannot reach consensus
```

**Диагностика:**

```bash
# Проверка статуса Raft
curl http://master-1:9333/cluster/raft?pretty=y

# Логи master серверов
sudo journalctl -u seaweedfs-master -n 100
```

**Решение:**

```bash
# Вариант 1: Перезапуск master серверов по очереди
for HOST in master-1 master-2 master-3; do
    ssh $HOST "sudo systemctl restart seaweedfs-master"
    sleep 10
done

# Вариант 2: Принудительный reset Raft (ОСТОРОЖНО!)
weed shell -master=master-1:9333 << EOF
cluster.raft.reset
EOF

# Вариант 3: Удаление поврежденных метаданных и восстановление
sudo systemctl stop seaweedfs-master
sudo rm -rf /data/seaweedfs/metadata/raft
sudo systemctl start seaweedfs-master
```

#### Проблема 2: Volume сервер не подключается к Master

**Симптомы:**
```
Error: cannot connect to master
Volume server not visible in topology
```

**Диагностика:**

```bash
# Проверка доступности master
curl http://master-1:9333/cluster/status

# Проверка логов volume
sudo journalctl -u seaweedfs-volume -n 100 | grep -i error

# Проверка сети
ping master-1
telnet master-1 9333
```

**Решение:**

```bash
# Проверка firewall
sudo firewall-cmd --list-all

# Открытие портов если нужно
sudo firewall-cmd --permanent --add-port=9333/tcp
sudo firewall-cmd --reload

# Проверка параметров запуска
sudo systemctl cat seaweedfs-volume

# Перезапуск с правильными параметрами
sudo systemctl restart seaweedfs-volume
```

#### Проблема 3: Filer не может подключиться к базе данных

**Симптомы:**
```
Error: connection to PostgreSQL failed
Database timeout errors
```

**Диагностика:**

```bash
# Проверка подключения к PostgreSQL
psql -h postgres-host -U seaweedfs -d seaweedfs -c "SELECT 1;"

# Проверка логов filer
sudo journalctl -u seaweedfs-filer -f

# Проверка connections в PostgreSQL
psql -h postgres-host -U postgres -c "SELECT count(*) FROM pg_stat_activity WHERE datname='seaweedfs';"
```

**Решение:**

```bash
# Увеличение max_connections в PostgreSQL
sudo vim /etc/postgresql/14/main/postgresql.conf
# max_connections = 200

# Перезапуск PostgreSQL
sudo systemctl restart postgresql

# Очистка idle connections
psql -h postgres-host -U postgres -c "
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname='seaweedfs' 
AND state='idle' 
AND state_change < now() - interval '5 minutes';
"

# Настройка connection pooling (PgBouncer)
```

#### Проблема 4: Низкая производительность записи

**Симптомы:**
```
Slow upload speeds
High latency on writes
Timeouts during uploads
```

**Диагностика:**

```bash
# Проверка I/O на volume серверах
iostat -x 5

# Проверка disk wait
top
# Смотреть на %wa (wait)

# Проверка метрик volume
curl http://volume-1-1:18080/metrics | grep weed_volume

# Проверка сети
iftop -i eth0
```

**Решение:**

```bash
# 1. Оптимизация I/O scheduler
echo "mq-deadline" | sudo tee /sys/block/sdb/queue/scheduler

# 2. Увеличение компрессии
# В systemd service:
-compactionMBps=200

# 3. Проверка репликации
# Снижение репликации для некритичных данных
# 010 -> 000

# 4. Добавление volume серверов
# Увеличение параллелизма записи

# 5. Tune filesystem
sudo mount -o remount,noatime,nodiratime /data/disk1
```

#### Проблема 5: Заполнение дискового пространства

**Симптомы:**
```
Error: no writable volumes
Disk space full errors
Cannot create new volumes
```

**Диагностика:**

```bash
# Проверка свободного места
df -h | grep seaweedfs

# Проверка volumes
weed shell -master=master-1:9333 << EOF
volume.list
EOF

# Проверка старых deleted volumes
find /data/disk*/seaweedfs/volumes -name "*.dat" -ls
```

**Решение:**

```bash
# 1. Vacuum - удаление deleted файлов
weed shell -master=master-1:9333 << EOF
volume.vacuum -garbageThreshold=0.3
EOF

# 2. Принудительная компрессия
weed shell -master=master-1:9333 << EOF
volume.vacuum -force
EOF

# 3. Перемещение volumes на другие диски
weed shell -master=master-1:9333 << EOF
volume.move -source=volume-1-1:8080 -target=volume-2-1:8080 -volumeId=123
EOF

# 4. Добавление новых дисков/серверов

# 5. Удаление старой коллекции (если применимо)
weed shell -master=master-1:9333 << EOF
collection.delete -collection=old-data
EOF
```

#### Проблема 6: Потеря реплик

**Симптомы:**
```
Replica count mismatch
Missing replicas warnings
Data unavailable errors
```

**Диагностика:**

```bash
# Проверка репликации
weed shell -master=master-1:9333 << EOF
volume.check.disk
volume.check.replication
EOF

# Проверка конкретного volume
curl "http://master-1:9333/dir/lookup?volumeId=123&pretty=y"
```

**Решение:**

```bash
# Автоматическое восстановление реплик
weed shell -master=master-1:9333 << EOF
volume.fix.replication
EOF

# Принудительная репликация
weed shell -master=master-1:9333 << EOF
volume.fix.replication -force
EOF

# Балансировка после восстановления
weed shell -master=master-1:9333 << EOF
volume.balance -force
EOF

# Если проблема персистит - проверка volume integrity
weed shell -master=master-1:9333 << EOF
volume.fsck -volumeId=123
EOF
```

#### Проблема 7: S3 API authentication failures

**Симптомы:**
```
403 Forbidden
SignatureDoesNotMatch
InvalidAccessKeyId
```

**Диагностика:**

```bash
# Проверка S3 config
cat /etc/seaweedfs/s3.json

# Тест подключения
aws s3 ls --endpoint-url http://filer-1:8333 --debug

# Проверка логов
sudo journalctl -u seaweedfs-filer | grep -i s3
```

**Решение:**

```bash
# 1. Проверка credentials в s3.json
sudo vim /etc/seaweedfs/s3.json

# 2. Перезапуск filer после изменений
sudo systemctl restart seaweedfs-filer

# 3. Проверка прав доступа к файлу
sudo chmod 600 /etc/seaweedfs/s3.json
sudo chown seaweedfs:seaweedfs /etc/seaweedfs/s3.json

# 4. Тест с curl
curl -X PUT http://filer-1:8333/test-bucket \
  -H "Authorization: AWS access_key:$(echo -n 'PUT\n\n\n\n/test-bucket' | \
  openssl dgst -sha1 -hmac 'secret_key' -binary | base64)"
```

### Diagnostic scripts

**Comprehensive health check:**

```bash
#!/bin/bash
# comprehensive-check.sh

echo "=== SeaweedFS Comprehensive Diagnostic ==="
echo "Date: $(date)"
echo ""

# 1. Master check
echo "### MASTER SERVERS ###"
for HOST in master-1 master-2 master-3; do
    echo "Checking $HOST..."
    if curl -sf "http://$HOST:9333/cluster/status" > /dev/null 2>&1; then
        STATUS=$(curl -s "http://$HOST:9333/cluster/status" | jq -r '.IsLeader')
        echo "  ✓ $HOST is $([ "$STATUS" = "true" ] && echo 'LEADER' || echo 'FOLLOWER')"
    else
        echo "  ❌ $HOST is DOWN"
    fi
done

# 2. Volume check
echo ""
echo "### VOLUME SERVERS ###"
VOLUMES=$(curl -s "http://master-1:9333/dir/status" | \
    jq -r '.Topology.DataCenters[].Racks[].DataNodes[] | "\(.Url) \(.VolumeCount) \(.MaxVolumeCount)"')
echo "$VOLUMES" | while read URL CURRENT MAX; do
    USAGE=$((CURRENT * 100 / MAX))
    if [ $USAGE -gt 80 ]; then
        echo "  ⚠️  $URL: ${CURRENT}/${MAX} volumes (${USAGE}%)"
    else
        echo "  ✓ $URL: ${CURRENT}/${MAX} volumes (${USAGE}%)"
    fi
done

# 3. Filer check
echo ""
echo "### FILER SERVERS ###"
for HOST in filer-1 filer-2 filer-3; do
    if curl -sf "http://$HOST:8888/" > /dev/null 2>&1; then
        echo "  ✓ $HOST is UP"
    else
        echo "  ❌ $HOST is DOWN"
    fi
done

# 4. Replication check
echo ""
echo "### REPLICATION STATUS ###"
ISSUES=$(curl -s "http://master-1:9333/vol/status" | \
    jq -r '.Volumes[] | select(.ReplicaPlacement != .FileCount) | .Id' | wc -l)
if [ "$ISSUES" -eq 0 ]; then
    echo "  ✓ All volumes properly replicated"
else
    echo "  ❌ $ISSUES volumes with replication issues"
fi

# 5. Disk space check
echo ""
echo "### DISK SPACE ###"
TOTAL=$(curl -s "http://master-1:9333/dir/status" | jq -r '.Topology.Max')
FREE=$(curl -s "http://master-1:9333/dir/status" | jq -r '.Topology.Free')
USED=$((TOTAL - FREE))
USAGE=$((USED * 100 / TOTAL))

if [ $USAGE -gt 90 ]; then
    echo "  ❌ Critical: ${USAGE}% used"
elif [ $USAGE -gt 80 ]; then
    echo "  ⚠️  Warning: ${USAGE}% used"
else
    echo "  ✓ OK: ${USAGE}% used"
fi

echo ""
echo "=== End of Diagnostic ==="
```

```bash
chmod +x /usr/local/bin/comprehensive-check.sh
```

### Debug mode и логирование

**Включение debug режима:**

```bash
# Для master
sudo systemctl edit seaweedfs-master
# Добавить:
[Service]
Environment="WEED_DEBUG=true"

# Для volume
sudo systemctl edit seaweedfs-volume
[Service]
Environment="WEED_DEBUG=true"

# Для filer
sudo systemctl edit seaweedfs-filer
[Service]
Environment="WEED_DEBUG=true"

# Перезапуск
sudo systemctl daemon-reload
sudo systemctl restart seaweedfs-master seaweedfs-volume seaweedfs-filer
```

**Увеличение verbosity логов:**

```bash
# Добавить параметр -v в ExecStart
-v=3  # Level 3 - detailed logging
-v=4  # Level 4 - very detailed
```

### Network troubleshooting

**Проверка связности между серверами:**

```bash
#!/bin/bash
# network-check.sh

MASTERS=(master-1:9333 master-2:9333 master-3:9333)
VOLUMES=(volume-1-1:8080 volume-2-1:8080 volume-3-1:8080)
FILERS=(filer-1:8888 filer-2:8888 filer-3:8888)

echo "=== Network Connectivity Check ==="

# Check masters
echo "Checking masters..."
for MASTER in "${MASTERS[@]}"; do
    if timeout 5 bash -c "echo > /dev/tcp/${MASTER/:/ }" 2>/dev/null; then
        echo "  ✓ $MASTER reachable"
    else
        echo "  ❌ $MASTER unreachable"
    fi
done

# Check volumes
echo "Checking volumes..."
for VOLUME in "${VOLUMES[@]}"; do
    if timeout 5 bash -c "echo > /dev/tcp/${VOLUME/:/ }" 2>/dev/null; then
        echo "  ✓ $VOLUME reachable"
    else
        echo "  ❌ $VOLUME unreachable"
    fi
done

# Check filers
echo "Checking filers..."
for FILER in "${FILERS[@]}"; do
    if timeout 5 bash -c "echo > /dev/tcp/${FILER/:/ }" 2>/dev/null; then
        echo "  ✓ $FILER reachable"
    else
        echo "  ❌ $FILER unreachable"
    fi
done
```

### Полезные команды для troubleshooting

```bash
# Проверка открытых файлов
lsof -u seaweedfs | wc -l

# Проверка сетевых соединений
netstat -an | grep -E ':(9333|8080|8888|8333)'

# Проверка процессов
ps aux | grep weed

# Проверка использования ресурсов
top -u seaweedfs

# Trace системных вызовов
strace -p $(pidof weed) -e trace=network

# Проверка DNS разрешения
nslookup master-1
dig master-1

# Trace HTTP запросов
tcpdump -i eth0 -s 0 -A 'tcp port 8080'
```

---

## Production чеклист

### Pre-deployment чеклист

#### Инфраструктура
- [ ] Минимум 3 master сервера для HA
- [ ] Минимум 6 volume серверов (2 на rack)
- [ ] Минимум 2 filer сервера
- [ ] Балансировщики нагрузки настроены (Nginx/HAProxy)
- [ ] Firewall правила настроены
- [ ] DNS записи созданы
- [ ] SSL сертификаты установлены (для S3/Filer API)

#### Хранение
- [ ] Диски отформатированы в XFS
- [ ] Mount options оптимизированы (noatime, nodiratime)
- [ ] I/O scheduler настроен правильно
- [ ] Минимум 10% свободного места на дисках
- [ ] RAID не используется (SeaweedFS сам управляет репликацией)

#### Настройки системы
- [ ] Kernel parameters оптимизированы (sysctl.conf)
- [ ] File descriptors увеличены (ulimit)
- [ ] Swap отключен или минимален (swappiness=10)
- [ ] NTP синхронизация настроена
- [ ] Timezone корректен на всех серверах

#### Конфигурация SeaweedFS
- [ ] Master peers настроены корректно
- [ ] Репликация настроена (010 или выше)
- [ ] Volume size оптимален (30GB)
- [ ] Rack/DC awareness настроен
- [ ] Filer база данных (PostgreSQL) настроена
- [ ] S3 credentials созданы
- [ ] Collection strategy определена

#### Безопасность
- [ ] Пользователь seaweedfs создан (non-root)
- [ ] Права доступа к файлам настроены (750/600)
- [ ] Firewall включен и настроен
- [ ] SELinux/AppArmor настроен (если используется)
- [ ] Сеть изолирована (private VLAN)
- [ ] TLS включен для внешних API

#### Мониторинг
- [ ] Prometheus установлен и настроен
- [ ] Node exporter установлен на всех серверах
- [ ] Grafana dashboards импортированы
- [ ] Alert rules настроены
- [ ] Alertmanager настроен (email/Slack/PagerDuty)
- [ ] Health check скрипты созданы

#### Backup
- [ ] Master metadata backup настроен
- [ ] Filer database backup настроен
- [ ] Backup retention policy определена
- [ ] Restore процедура протестирована
- [ ] Offsite backup настроен

#### Документация
- [ ] Network diagram создан
- [ ] Server inventory задокументирован
- [ ] Runbook создан для операций
- [ ] Disaster recovery план написан
- [ ] Контакты on-call команды определены

### Post-deployment чеклист

#### Функциональное тестирование
- [ ] Master cluster выбрал лидера
- [ ] Все volume серверы подключены
- [ ] Filer доступен и работает
- [ ] S3 API отвечает
- [ ] Загрузка файлов работает
- [ ] Скачивание файлов работает
- [ ] Репликация работает корректно
- [ ] Балансировщик распределяет нагрузку

#### Performance тестирование
- [ ] Upload benchmark выполнен
- [ ] Download benchmark выполнен
- [ ] Latency в пределах SLA
- [ ] Throughput соответствует ожиданиям
- [ ] Concurrent connections тест пройден

#### Отказоустойчивость
- [ ] Failover master протестирован
- [ ] Потеря volume сервера не ломает систему
- [ ] Потеря filer не ломает систему
- [ ] Восстановление после перезагрузки работает
- [ ] Репликация восстанавливается автоматически

#### Мониторинг
- [ ] Метрики собираются корректно
- [ ] Dashboards отображают данные
- [ ] Alerts срабатывают
- [ ] Логи централизованно собираются
- [ ] Health checks проходят

#### Операционная готовность
- [ ] Команда обучена управлению кластером
- [ ] Runbook актуален
- [ ] On-call rotation настроен
- [ ] Эскалация процессов определена
- [ ] Backup/restore протестированы

### Monthly maintenance чеклист

- [ ] Проверка disk space (должно быть >20% свободно)
- [ ] Проверка replication integrity
- [ ] Vacuum старых deleted файлов
- [ ] Проверка backup (restore test)
- [ ] Обновление OS security patches
- [ ] Проверка logs на errors/warnings
- [ ] Review Grafana dashboards
- [ ] Проверка certificate expiration (SSL)
- [ ] Performance benchmark
- [ ] Capacity planning review

### Quarterly maintenance чеклист

- [ ] SeaweedFS version upgrade (если доступно)
- [ ] PostgreSQL upgrade (minor version)
- [ ] OS upgrade (если требуется)
- [ ] Disaster recovery drill
- [ ] Security audit
- [ ] Performance tuning review
- [ ] Capacity expansion planning
- [ ] Documentation update
- [ ] Team training refresh

### Upgrade процедура

**Безопасное обновление SeaweedFS:**

```bash
#!/bin/bash
# upgrade-seaweedfs.sh

NEW_VERSION="3.60"
BACKUP_DIR="/backup/seaweedfs/upgrade-$(date +%Y%m%d)"

echo "=== SeaweedFS Upgrade to $NEW_VERSION ==="
echo "Date: $(date)"
echo ""

# 1. Создание backup
echo "Step 1: Creating backups..."
mkdir -p "$BACKUP_DIR"
/usr/local/bin/backup-master.sh
/usr/local/bin/backup-filer-db.sh

# 2. Скачивание новой версии
echo "Step 2: Downloading new version..."
cd /tmp
wget "https://github.com/seaweedfs/seaweedfs/releases/download/$NEW_VERSION/linux_amd64.tar.gz"
tar -xzf linux_amd64.tar.gz

# 3. Backup текущего бинарного файла
echo "Step 3: Backing up current binary..."
sudo cp /usr/local/bin/weed "$BACKUP_DIR/weed.old"

# 4. Установка нового бинарного файла
echo "Step 4: Installing new binary..."
sudo cp weed /usr/local/bin/weed
sudo chmod +x /usr/local/bin/weed
sudo chown root:root /usr/local/bin/weed

# 5. Проверка версии
echo "Step 5: Verifying new version..."
NEW_VER=$(weed version | grep -oP 'version \K[0-9.]+')
echo "Installed version: $NEW_VER"

# 6. Rolling restart - Filer (non-critical)
echo "Step 6: Restarting filer servers..."
for HOST in filer-1 filer-2 filer-3; do
    echo "  Restarting $HOST..."
    ssh $HOST "sudo systemctl restart seaweedfs-filer"
    sleep 30
    
    # Проверка здоровья
    if ssh $HOST "sudo systemctl is-active --quiet seaweedfs-filer"; then
        echo "  ✓ $HOST is healthy"
    else
        echo "  ❌ $HOST failed to start!"
        exit 1
    fi
done

# 7. Rolling restart - Volume servers
echo "Step 7: Restarting volume servers..."
for HOST in volume-1-1 volume-1-2 volume-2-1 volume-2-2 volume-3-1 volume-3-2; do
    echo "  Restarting $HOST..."
    ssh $HOST "sudo systemctl restart seaweedfs-volume"
    sleep 60  # Больше времени для volumes
    
    if ssh $HOST "sudo systemctl is-active --quiet seaweedfs-volume"; then
        echo "  ✓ $HOST is healthy"
    else
        echo "  ❌ $HOST failed to start!"
        exit 1
    fi
done

# 8. Rolling restart - Master servers (critical!)
echo "Step 8: Restarting master servers..."
for HOST in master-2 master-3 master-1; do  # Leader последним
    echo "  Restarting $HOST..."
    ssh $HOST "sudo systemctl restart seaweedfs-master"
    sleep 60
    
    # Проверка Raft consensus
    if curl -sf "http://$HOST:9333/cluster/status" > /dev/null; then
        echo "  ✓ $HOST is healthy"
    else
        echo "  ❌ $HOST failed to start!"
        exit 1
    fi
done

# 9. Final health check
echo "Step 9: Final health check..."
/usr/local/bin/check-cluster-health.sh

echo ""
echo "✓ Upgrade completed successfully!"
echo "New version: $NEW_VER"
```

### Emergency procedures

**Процедура экстренного восстановления:**

```markdown
# Emergency Response Procedures

## Полная потеря Master кластера

1. **Stop все компоненты**
   ```bash
   for HOST in volume-* filer-*; do
       ssh $HOST "sudo systemctl stop seaweedfs-*"
   done
   ```

2. **Восстановить master из последнего backup**
   ```bash
   for HOST in master-1 master-2 master-3; do
       scp /backup/master-metadata-latest.tar.gz $HOST:/tmp/
       ssh $HOST "/usr/local/bin/restore-master.sh /tmp/master-metadata-latest.tar.gz"
   done
   ```

3. **Запустить master кластер**
   ```bash
   for HOST in master-1 master-2 master-3; do
       ssh $HOST "sudo systemctl start seaweedfs-master"
       sleep 30
   done
   ```

4. **Проверить выбор лидера**
   ```bash
   curl http://master-1:9333/cluster/status?pretty=y
   ```

5. **Запустить volume и filer серверы**
   ```bash
   for HOST in volume-* filer-*; do
       ssh $HOST "sudo systemctl start seaweedfs-*"
   done
   ```

## Потеря базы данных Filer

1. **Stop все filer серверы**
2. **Восстановить PostgreSQL из backup**
3. **Restart filer серверы**
4. **Verify filesystem через S3 API**

## Критическая нехватка места

1. **Немедленно запустить vacuum**
   ```bash
   weed shell -master=master-1:9333 << EOF
   volume.vacuum -garbageThreshold=0.3 -force
   EOF
   ```

2. **Добавить экстренный volume сервер**

3. **Migrate данные с полных дисков**

4. **Plan capacity expansion**

## Контакты для эскалации

- **L1 Support**: team@example.com
- **L2 Support**: oncall@example.com
- **L3 Support**: senior-eng@example.com
- **Emergency**: +1-XXX-XXX-XXXX
```

### Capacity planning

**Формулы для планирования роста:**

```python
#!/usr/bin/env python3
# capacity-planner.py

def calculate_capacity(
    num_volumes_servers,
    disks_per_server,
    disk_size_tb,
    replication_factor,
    growth_rate_percent,
    months_ahead
):
    """
    Рассчитывает capacity и потребности в расширении
    """
    # Raw capacity
    raw_capacity_tb = num_volumes_servers * disks_per_server * disk_size_tb
    
    # Usable capacity с учетом репликации
    usable_capacity_tb = raw_capacity_tb / replication_factor
    
    # Предполагаемый рост
    monthly_growth = usable_capacity_tb * (growth_rate_percent / 100)
    projected_usage = usable_capacity_tb * 0.7  # Текущее 70%
    
    months_until_full = (usable_capacity_tb * 0.9 - projected_usage) / monthly_growth
    
    print(f"=== Capacity Planning Report ===")
    print(f"Current Configuration:")
    print(f"  Volume Servers: {num_volumes_servers}")
    print(f"  Disks per Server: {disks_per_server}")
    print(f"  Disk Size: {disk_size_tb}TB")
    print(f"  Replication Factor: {replication_factor}")
    print(f"")
    print(f"Capacity:")
    print(f"  Raw Capacity: {raw_capacity_tb:.1f}TB")
    print(f"  Usable Capacity: {usable_capacity_tb:.1f}TB")
    print(f"  Current Usage (70%): {projected_usage:.1f}TB")
    print(f"")
    print(f"Growth Projection:")
    print(f"  Monthly Growth Rate: {growth_rate_percent}%")
    print(f"  Monthly Growth: {monthly_growth:.1f}TB")
    print(f"  Months Until 90% Full: {months_until_full:.1f}")
    print(f"")
    
    if months_until_full < 6:
        print(f"⚠️  WARNING: Capacity will be exhausted in {months_until_full:.1f} months!")
        print(f"   Action required: Plan expansion immediately")
    elif months_until_full < 12:
        print(f"⚠️  NOTICE: Capacity sufficient for {months_until_full:.1f} months")
        print(f"   Action required: Start planning expansion")
    else:
        print(f"✓ Capacity sufficient for {months_until_full:.1f} months")
    
    # Рекомендации по расширению
    if months_until_full < 12:
        additional_servers = int((monthly_growth * 12) / (disks_per_server * disk_size_tb / replication_factor)) + 1
        print(f"")
        print(f"Expansion Recommendation:")
        print(f"  Add {additional_servers} volume servers")
        print(f"  This will provide capacity for ~12 months")

# Пример использования
if __name__ == "__main__":
    calculate_capacity(
        num_volumes_servers=9,
        disks_per_server=8,
        disk_size_tb=6,
        replication_factor=2,
        growth_rate_percent=5,  # 5% месячный рост
        months_ahead=12
    )
```

### Final recommendations

**Best practices для production:**

1. **Всегда используйте нечетное количество master серверов** (3, 5, 7)
2. **Минимум 2 копии данных** (репликация 010 или выше)
3. **Rack awareness обязателен** для production
4. **Мониторинг 24/7** с алертами
5. **Автоматизированные backup** ежедневно
6. **Тестирование restore** ежемесячно
7. **Capacity planning** ежеквартально
8. **Rolling upgrades** для zero-downtime
9. **Документация актуальна** всегда
10. **Disaster recovery plan** протестирован

**Типичные ошибки, которых следует избегать:**

- ❌ Использование RAID под SeaweedFS
- ❌ Одинаковые volume серверы в одном rack без репликации
- ❌ Отсутствие мониторинга
- ❌ Отсутствие backup метаданных
- ❌ Игнорирование disk space warnings
- ❌ Upgrade всех серверов одновременно
- ❌ Использование репликации 000 в production
- ❌ Недостаточное тестирование перед production
- ❌ Отсутствие capacity planning
- ❌ Single point of failure в инфраструктуре

---

## Заключение

Этот гайд покрывает все аспекты установки и эксплуатации SeaweedFS в production окружении. Следуя этим рекомендациям, вы получите:

- **Высокую доступность**: 99.9%+ uptime с правильной репликацией
- **Масштабируемость**: от сотен миллионов до миллиардов файлов
- **Производительность**: низкая latency и высокий throughput
- **Надежность**: защита данных и быстрое восстановление
- **Управляемость**: простая эксплуатация и мониторинг

### Дополнительные ресурсы

- **Официальная документация**: https://github.com/seaweedfs/seaweedfs/wiki
- **Community форум**: https://github.com/seaweedfs/seaweedfs/discussions
- **Issue tracker**: https://github.com/seaweedfs/seaweedfs/issues
- **Slack community**: https://seaweedfs.slack.com
- **Examples**: https://github.com/seaweedfs/seaweedfs/tree/master/examples

### Поддержка

Для получения помощи:
- GitHub Issues для bug reports
- GitHub Discussions для вопросов
- Enterprise support от SeaweedFS команды
- Community Slack для быстрых вопросов

**Успешного развертывания! 🚀**
