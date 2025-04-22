# 4

**Пошаговое руководство по настройке мониторинга:**

### 1. Создание инстанса Cloud-MON
- **Параметры инстанса**:
  - Тип: 1 vCPU, 1 ГБ RAM.
  - Диск: 10 ГБ.
  - ОС: Альт Starterkit (шаблон `alt-p10-cloud-x86_64`).

### 2. Настройка SSH-доступа с CloudADM на Cloud-MON
1. **Генерация SSH-ключа на CloudADM** (если отсутствует):
   ```bash
   ssh-keygen -t rsa -f ~/.ssh/id_rsa_cloud_mon
   ```
2. **Добавление публичного ключа на Cloud-MON**:
   - Скопируйте содержимое `~/.ssh/id_rsa_cloud_mon.pub` с CloudADM.
   - На Cloud-MON (под пользователем `altlinux`):
     ```bash
     mkdir -p ~/.ssh
     echo "ВАШ_ПУБЛИЧНЫЙ_КЛЮЧ" >> ~/.ssh/authorized_keys
     chmod 700 ~/.ssh
     chmod 600 ~/.ssh/authorized_keys
     ```
3. **Настройка SSH-конфига на CloudADM**:
   Добавьте в `~/.ssh/config`:
   ```config
   Host cloud-mon
       HostName <IP_Cloud-MON>
       User altlinux
       IdentityFile ~/.ssh/id_rsa_cloud_mon
   ```
   Проверьте подключение:
   ```bash
   ssh cloud-mon
   ```

### 3. Установка Docker и Docker Compose на Cloud-MON
```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose
sudo usermod -aG docker altlinux
# Перезайдите в сессию для применения прав
exit
ssh cloud-mon  # Повторное подключение
```

### 4. Создание Docker Compose-файла (`monitoring.yml`)
Создайте файл `~/monitoring.yml`:
```yaml
version: '3'
services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
    depends_on:
      - node-exporter

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana-storage:/var/lib/grafana
    ports:
      - 3000:3000
    depends_on:
      - prometheus

volumes:
  grafana-storage:
```

### 5. Настройка Prometheus (`prometheus.yml`)
Создайте файл `~/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node-mon'
    static_configs:
      - targets: ['node-exporter:9100']
  - job_name: 'cloud-adm'
    static_configs:
      - targets: ['Cloud-ADM:9100']
  - job_name: 'backup-nodes'
    static_configs:
      - targets: ['Cloud-BACKUP01:9100', 'Cloud-BACKUP02:9100']
```

### 6. Запуск сервисов
```bash
docker-compose -f monitoring.yml up -d
```

### 7. Настройка DNS на CloudADM
Добавьте в `/etc/hosts` на CloudADM:
```
<IP_Cloud-MON> grafana.au.team
```

### 8. Настройка Grafana
1. **Доступ к Grafana**:
   - Откройте `http://grafana.au.team:3000` с CloudADM.
   - Логин/пароль по умолчанию: `admin/admin`.

2. **Добавление источника данных**:
   - Перейдите в **Configuration > Data Sources > Add data source**.
   - Выберите **Prometheus**.
   - URL: `http://prometheus:9090` (внутри Docker-сети).

3. **Импорт дашборда**:
   - Перейдите в **Create > Import**.
   - Используйте ID дашборда **1860** (Node Exporter Full) или создайте вручную.

### 9. Настройка Node Exporter на остальных инстансах
Для **Cloud-ADM**, **Cloud-BACKUP01**, **Cloud-BACKUP02**:
```bash
# На каждом инстансе:
docker run -d --name node-exporter \
  -p 9100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  prom/node-exporter:latest \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys
```

### 10. Проверка работы
- **Prometheus Targets**: Откройте `http://cloud-mon:9090/targets` — все цели должны быть `UP`.
- **Grafana Dashboard**: Убедитесь, что метрики CPU, RAM и Disk отображаются для всех узлов.

**Готово!** Система мониторинга настроена.
