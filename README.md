# Установка и настройка кластера OpenSearch 2.12

> Версия: **OpenSearch 2.12**  
> Кластер: **3 master node + 2 data node**  
> Действия выполняются от пользователя `user`.

---

## 📦 Установка

### Скачиваем и распаковываем архив

```bash
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.12.0/opensearch-2.12.0-linux-x64.tar.gz
sudo chmod +x opensearch-2.12.0-linux-x64.tar.gz
sudo tar -xf opensearch-2.12.0-linux-x64.tar.gz
sudo mkdir /opt/opensearch
sudo mv ./opensearch-2.12.0/* /opt/opensearch
sudo chown -R user:user /opt/opensearch
cd /opt/opensearch
```

---

## 🔐 Настройка администратора

### Устанавливаем пароль

```bash
export OPENSEARCH_INITIAL_ADMIN_PASSWORD=<custom-admin-password>
```

### Запускаем скрипт

```bash
./opensearch-tar-install.sh
```

### Проверяем работу

```bash
curl -X GET https://localhost:9200 -u 'admin:<custom-admin-password>' --insecure
```

### Проверка плагинов

```bash
curl -X GET https://localhost:9200/_cat/plugins?v -u 'admin:<custom-admin-password>' --insecure
```

---

## ⚙️ Конфигурация

### `opensearch.yml`

```yaml
network.host: 0.0.0.0
discovery.type: single-node
plugins.security.disabled: false
```

### `jvm.options`

```bash
-Xms4g
-Xmx4g
```

### Переменная окружения

```bash
export OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk
```

---

## 🔒 TLS-сертификаты

Создаем скрипт `add_cert.sh` в `/opt/opensearch/config` с генерацией сертификатов (root, admin, node1, node2, client).  
После создания:

```bash
chmod +x ./add_cert.sh
./add_cert.sh
```

### Настройка `opensearch.yml` для TLS

```yaml
plugins.security.ssl.transport.pemcert_filepath: node1.pem
plugins.security.ssl.transport.pemkey_filepath: node1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false

plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: node1.pem
plugins.security.ssl.http.pemkey_filepath: node1-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: root-ca.pem

plugins.security.authcz.admin_dn:
  - 'CN=A,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'

plugins.security.nodes_dn:
  - 'CN=node1.dns.a-record,...'
  - 'CN=node2.dns.a-record,...'
```

---

## 👤 Пользователь `admin`

### Генерация хэша

```bash
cd /opt/opensearch/plugins/opensearch-security/tools/
OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk ./hash.sh
```

### `internal_users.yml`

```yaml
admin:
  hash: "<ваш-хэш>"
  reserved: true
  backend_roles:
    - "admin"
  description: "Admin user"
```

---

## 🛡️ Инициализация безопасности

```bash
cd /opt/opensearch/plugins/opensearch-security/tools

OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk ./securityadmin.sh \
  -cd /opt/opensearch/config/opensearch-security/ \
  -cacert /opt/opensearch/config/root-ca.pem \
  -cert /opt/opensearch/config/admin.pem \
  -key /opt/opensearch/config/admin-key.pem \
  -icl -nhnv
```

---

## 🧩 systemd-сервис

### `opensearch.service`

```ini
[Unit]
Description=OpenSearch
Wants=network-online.target
After=network-online.target

[Service]
Type=forking
WorkingDirectory=/opt/opensearch
ExecStart=/opt/opensearch/bin/opensearch -d
User=user
Group=user
StandardOutput=journal
StandardError=inherit
LimitNOFILE=65535
LimitNPROC=4096
LimitAS=infinity
LimitFSIZE=infinity
TimeoutStopSec=0
KillSignal=SIGTERM
KillMode=process
SendSIGKILL=no
SuccessExitStatus=143
TimeoutStartSec=75

[Install]
WantedBy=multi-user.target
```

### Активируем и запускаем

```bash
sudo systemctl daemon-reload
sudo systemctl enable opensearch.service
sudo systemctl start opensearch
sudo systemctl status opensearch
```

---

## 🧠 Настройка кластера

### Master Node (пример)

```yaml
cluster.name: my-cluster
node.name: master-node-01
node.roles: [ master, data ]
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: [ "10.0.0.1", "10.0.0.2" ]
cluster.initial_master_nodes: [ "master-node-01", "master-node-02", "master-node-03" ]
```

### Data Node (пример)

```yaml
node.name: data-node-01
node.roles: [ data, ingest ]
```

### Проверка

```bash
curl -XGET https://<ip>:9200/_cat/nodes?v -u 'admin:<password>' --insecure
```

### Очистка, если ошибка `data`-директории

```bash
rm -rf /opt/opensearch/data/*
```

---

## 📊 Установка OpenSearch Dashboard

После сборки кластера установите **OpenSearch Dashboard** для визуального доступа. Инструкция по установке доступна на [opensearch.org](https://opensearch.org/docs/latest/dashboards/).

---
