# Домашнее задание к занятию «Резервное копирование баз данных» - Савкин И.Н.

## Задание 1. Резервное копирование в финансовой компании

### 1.1. Нужно восстанавливать данные за предыдущий день (на 00:00 или конец дня)
**Подходящий вариант:**  
Полная (Full) физическая или логическая резервная копия, выполняемая **ежедневно в 23:55–00:05**.  
Рекомендуется:  
- MySQL → `mysqldump --single-transaction --quick --lock-tables=false` или XtraBackup (полный физический бэкап)  
- PostgreSQL → `pg_dumpall` или физический бэкап через `pg_basebackup`  
Точка восстановления (RPO) = до 24 часов.

### 1.2. Нужно восстанавливать данные за час до поломки
**Подходящий вариант:**  
Полный бэкап раз в сутки + **инкрементные/дифференциальные бэкапы + бинарные логи**.  
Схема:  
- Ежедневно в 00:00 — полный бэкап  
- Каждый час — инкрементный бэкап (MySQL: Percona XtraBackup --incremental, PostgreSQL: WAL-архивация + pg_receivewal)  
- Постоянно архивируются binlog (MySQL) или WAL (PostgreSQL)  

Точка восстановления (RPO) = до 5–15 минут (в зависимости от частоты архивации логов).

### 1.3.* Моментальное переключение при поломке (zero-downtime)

Варианты:  
- MySQL: Group Replication / InnoDB Cluster + MySQL Router (автоматический failover за 5–15 сек)  
- MySQL: полу-синхронная репликация + Orchestrator / MHA  
- PostgreSQL: Patroni + etcd/Consul/ZooKeeper (автоматический лидер за 5–10 сек)  
- PostgreSQL: репликация с hot standby + pg_auto_failover  

В этом случае при падении мастера приложение автоматически переключается на уже готовую реплику → downtime ≈ 0–15 секунд.

## Задание 2. PostgreSQL — pg_dump / pg_restore

### 2.1. Примеры команд (официальная документация 17)

```bash
# Полный логический бэкап всей инстанции
pg_dumpall -U postgres > full_backup_20251129.sql

# Бэкап отдельной базы в кастомном формате (самый быстрый для восстановления)
pg_dump -U postgres -Fc -b -v -f bookstore.backup bookstore

# Восстановление
pg_restore -U postgres -d bookstore -v --clean --create bookstore.backup
# или в новую пустую инстанцию
pg_restore -U postgres --clean --create -d postgres bookstore.backup
```

## Задание 3. MySQL — инкрементное копирование

### 3.1. Официальный и самый надёжный способ

```bash
# Полный бэкап (один раз в сутки)
xtrabackup --backup --target-dir=/backups/full_20251129 --user=backup --password=pass

# Инкрементный бэкап каждый час (на основе предыдущего)
xtrabackup --backup --target-dir=/backups/inc_20251129_0300 \
  --incremental-basedir=/backups/full_20251129

xtrabackup --backup --target-dir=/backups/inc_20251129_0400 \
  --incremental-basedir=/backups/inc_20251129_0300

# Подготовка к восстановлению
xtrabackup --prepare --target-dir=/backups/full_20251129
xtrabackup --prepare --apply-only-full-backup --target-dir=/backups/full_20251129
xtrabackup --prepare --target-dir=/backups/full_20251129 \
  --incremental-dir=/backups/inc_20251129_0400
```
