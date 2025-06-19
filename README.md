Установка и настройка кластера OpenSearch 2.12

Кластер состоит из 3 master node и 2 data node.
Все действия выполняются от пользователя user.
🔧 Подготовка

Скачивание и установка
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.12.0/opensearch-2.12.0-linux-x64.tar.gz
sudo chmod +x opensearch-2.12.0-linux-x64.tar.gz
sudo tar -xf opensearch-2.12.0-linux-x64.tar.gz
sudo mkdir /opt/opensearch
sudo mv ./opensearch-2.12.0/* /opt/opensearch
sudo chown -R user:user /opt/opensearch
cd /opt/opensearch
Установка переменной пароля администратора
export OPENSEARCH_INITIAL_ADMIN_PASSWORD=<custom-admin-password>
Замените <custom-admin-password> на сложный и уникальный пароль.
Запуск скрипта
./opensearch-tar-install.sh
Откройте второй терминал и проверьте:

curl -X GET https://localhost:9200 -u 'admin:<custom-admin-password>' --insecure
Ожидаемый результат:

{
  "name" : "hostname",
  "cluster_name" : "opensearch",
  ...
}
Проверка установленных плагинов
curl -X GET https://localhost:9200/_cat/plugins?v -u 'admin:<custom-admin-password>' --insecure
⚙️ Начальная настройка

Настройка opensearch.yml
sudo vim /opt/opensearch/config/opensearch.yml
Раскомментируйте:

network.host: 0.0.0.0
discovery.type: single-node
plugins.security.disabled: false
Настройка jvm.options
sudo vim /opt/opensearch/config/jvm.options
Установите:

-Xms4g
-Xmx4g
Обновление переменной окружения
export OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk
🔐 TLS сертификаты

cd /opt/opensearch/config/
vim add_cert.sh
Вставьте скрипт генерации сертификатов (см. выше в полном тексте).

Сделайте исполняемым и запустите:

chmod +x ./add_cert.sh
./add_cert.sh
Убедитесь, что на каждой ноде используются свои сертификаты, кроме root и admin.
Настройка opensearch.yml с сертификатами
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
👤 Пользователь admin

Генерация хэша пароля
chmod 755 /opt/opensearch/plugins/opensearch-security/tools/*.sh
cd /opt/opensearch/plugins/opensearch-security/tools/
OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk ./hash.sh
Сохраните сгенерированный хэш.

Обновление internal_users.yml
admin:
  hash: "<ваш-хэш>"
  reserved: true
  backend_roles:
    - "admin"
  description: "Admin user"
🛡️ Инициализация плагина безопасности

Запустите OpenSearch:
cd /opt/opensearch/bin
./opensearch
Во втором терминале выполните:
cd /opt/opensearch/plugins/opensearch-security/tools
OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk ./securityadmin.sh \
  -cd /opt/opensearch/config/opensearch-security/ \
  -cacert /opt/opensearch/config/root-ca.pem \
  -cert /opt/opensearch/config/admin.pem \
  -key /opt/opensearch/config/admin-key.pem \
  -icl -nhnv
🧩 Создание systemd-сервиса

sudo vim /etc/systemd/system/opensearch.service
Вставьте:

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
Активация сервиса
sudo systemctl daemon-reload
sudo systemctl enable opensearch.service
sudo systemctl start opensearch
sudo systemctl status opensearch
🧩 Конфигурация кластера

Настройка master-node-01
cluster.name: your-cluster-name
node.name: master-node-01
node.roles: [ master, data ]
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: [ "ip-node-1", "ip-node-2", ... ]
cluster.initial_master_nodes: [ "master-node-01", "master-node-02", "master-node-03" ]
❗ Первая мастер-нода должна запускаться с master и data.
Пример для data-ноды
node.name: data-node-01
node.roles: [ data, ingest ]
Проверка добавленных узлов
curl -XGET https://<ip>:9200/_cat/nodes?v -u 'admin:<password>' --insecure
Если возникают ошибки с "data", очистите директорию:

rm -rf /opt/opensearch/data/*
📊 OpenSearch Dashboard

Кластер собран. Следующий шаг — установка OpenSearch Dashboard для веб-интерфейса.





# Установка и настройка кластера OpenSearch

В проекте описывается вариант установки кластера Opensearch и Opensearch-dashboard.

[Полный гайд по ручной установки Opensearch в файле](/opensearch.md)

### Гайд по установки и настройки Opensearch в разработке! 
