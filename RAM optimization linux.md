# Шпаргалка по оптимизации VPS (Debian 12, 878 MB RAM)

Дата настройки: Октябрь 2026  
Цель: Предотвращение сбоев из-за OOM (Out Of Memory) и оптимизация работы Docker-контейнеров.

---

## 1. Настройка ZRAM и ядра Linux

### Конфигурация ZRAM (`/etc/default/zramswap`)
ZRAM сжимает оперативные данные на лету, создавая быстрый Swap в физической памяти.
- **Путь к файлу:** `/etc/default/zramswap`
- **Параметры:**
  ```bash
  ALGORITHM=lz4
  SIZE=512
  ```
- **Команды для управления:**
  ```bash
  sudo systemctl restart zramswap  # Перезапуск ZRAM
  sudo zramctl                      # Проверка статуса сжатия
  ```

### Настройка Swappiness (`/etc/sysctl.conf`)
Для ZRAM приоритет вытеснения устанавливается на максимум, чтобы ядро сжимало неактивный фоновый кэш.
- **Параметр в `/etc/sysctl.conf`:**
  ```ini
  vm.swappiness=100
  ```
- **Применение без перезагрузки:**
  ```bash
  sudo sysctl -p
  ```

---

## 2. Ограничения ресурсов Docker (mem_limit)

Суммарный жесткий лимит для всех контейнеров зафиксирован на уровне **512 MiB**, что гарантирует запас в **~366 MiB** физической памяти для ядра и системных служб.

| Контейнер | Жесткий лимит RAM | Swap лимит | Назначение / Особенности |
| :--- | :--- | :--- | :--- |
| **adguardhome** | `192m` | `384m` | DNS-фильтрация и кэширование |
| **teamspeak-database** | `128m` | `256m` | MariaDB (InnoDB буфер урезан) |
| **arcane-agent** | `128m` | `256m` | Служебный агент |
| **teamspeak-server** | `64m` | `128m` | Сервер голосовой связи |

### Применение лимитов "на лету" (без Compose):
```bash
docker update --memory 192m --memory-swap 384m adguardhome
docker update --memory 128m --memory-swap 256m arcane-agent
```

---

## 3. Конфигурация MariaDB (`docker-compose.yml`)

Для предотвращения разрастания памяти MariaDB в `docker-compose.yml` переданы оптимизированные флаги запуска:

```yaml
  db:
    container_name: teamspeak-database
    image: mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: <your_password>
      MYSQL_DATABASE: teamspeak
    volumes:
      - /data/teamspeak/db:/var/lib/mysql
    mem_limit: 128m
    memswap_limit: 256m
    command: >
      --innodb-buffer-pool-size=32M
      --performance-schema=OFF
      --key-buffer-size=8M
      --max-connections=20
```

---

## 4. Оптимизация AdGuard Home (через веб-интерфейс)

1. **Журнал запросов:**
   * *Настройки $\rightarrow$ Общие настройки $\rightarrow$ Сохранять журнал запросов:* **24 часа** (или 7 дней).
   * Выполнена очистка старого лога запросов.
2. **DNS-кэш:**
   * *Настройки $\rightarrow$ Настройки DNS $\rightarrow$ Размер кэша:* `1048576` (1 МБ).
   * Отключена функция *Optimistic caching*.

---

## 5. Команды для быстрого мониторинга

- **Просмотр потребления памяти контейнерами:**
  ```bash
  sudo docker stats --no-stream
  ```
- **Просмотр общей памяти и Swap:**
  ```bash
  free -h
  ```
- **Просмотр эффективности сжатия ZRAM:**
  ```bash
  sudo zramctl
  ```
- **Сброс файлового кэша ядра (при необходимости):**
  ```bash
  sudo sync; echo 3 | sudo tee /proc/sys/vm/drop_caches
  ```