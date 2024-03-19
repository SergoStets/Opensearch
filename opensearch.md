# Установка и настройка кластера OpenSearch
# Данная гайд был написан на версии opensearch 2.12. 
# Кластер будет состоять из 3 master node и 2 data node.
* При использовании меньшего колличества МАСТЕР нод кластер будет держаться только на 1 мастер ноде которая будет так же исполнять роль дата роли  
Все действия будут выполняться от пользователя user.

Скачаиваем актуальный архив:

wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.12.0/opensearch-2.12.0-linux-x64.tar.gz

Даем права на выполнение для архива:

sudo chmod +x opensearch-2.12.0-linux-x64.tar.gz

Распаковываем архив:

sudo tar -xf opensearch-2.12.0-linux-x64.tar.gz

Будем устанавливать OpenSearch в каталог «/opt/opensearch», поэтому создаем рабочий каталог для OpenSearch:

sudo mkdir /opt/opensearch

Переносим распакованные данные в рабочий каталог:

sudo mv ./opensearch-2.12.0/* /opt/opensearch

# Мы будем использовать пользователя user для запуска сервиса!

sudo chown -R user:user /opt/opensearch

Переходим в директорию 

cd /opt/opensearch

Обьявляем переменное окружение:

export OPENSEARCH_INITIAL_ADMIN_PASSWORD=<custom-admin-password> #Замените на свой пароль СЛОЖНЫЙ!

Запускаем скрипт от пользователя user 

./opensearch-tar-install.sh

Открываем второй терминал и подключаемся к серверу:
Проверяем работоспособность машины 

curl -X GET https://localhost:9200 -u 'admin:<custom-admin-password>' --insecure

Должны получить:
 {
    "name" : "hostname",
    "cluster_name" : "opensearch",
    "cluster_uuid" : "6XNc9m2gTUSIoKDqJit0PA",
    "version" : {
       "distribution" : "opensearch",
       "number" : <version>,
       "build_type" : <build-type>,
       "build_hash" : <build-hash>,
       "build_date" : <build-date>,
       "build_snapshot" : false,
       "lucene_version" : <lucene-version>,
       "minimum_wire_compatibility_version" : "7.10.0",
       "minimum_index_compatibility_version" : "7.0.0"
    },
    "tagline" : "The OpenSearch Project: https://opensearch.org/"
 }


Проверяем установленые плагины:
curl -X GET https://localhost:9200/_cat/plugins?v -u 'admin:<custom-admin-password>' --insecure

Должны получить:
 name     component                            version
 hostname opensearch-alerting                  2.12.0
 hostname opensearch-anomaly-detection         2.12.0
 hostname opensearch-asynchronous-search       2.12.0
 hostname opensearch-cross-cluster-replication 2.12.0
 hostname opensearch-index-management          2.12.0
 hostname opensearch-job-scheduler             2.12.0
 hostname opensearch-knn                       2.12.0
 hostname opensearch-ml                        2.12.0
 hostname opensearch-notifications             2.12.0
 hostname opensearch-notifications-core        2.12.0
 hostname opensearch-observability             2.12.0
 hostname opensearch-performance-analyzer      2.12.0
 hostname opensearch-reports-scheduler         2.12.0
 hostname opensearch-security                  2.12.0
 hostname opensearch-sql                       2.12.0


После того как мы увидели что сервер работает отсанавливаем Ctrl+C 

-----------------------------------------------------------------------


После нам надо сделать начальную настройку ноды для иницилизации плагина безопасности и настроем потребление оперативной памяти. 

Редоктируем файл конфигурации: 
# Я использую Vim для редоктирования конфигов, вы можете использовать nano или тот редоктор который вам удобен! 

sudo vim /opt/opensearch/config/opensearch.yml

Раскоментируем следующие поля:

# Bind OpenSearch to the correct network interface. Use 0.0.0.0
# to include all available interfaces or specify an IP address
# assigned to a specific interface.
network.host: 0.0.0.0

# Unless you have already configured a cluster, you should set
# discovery.type to single-node, or the bootstrap checks will
# fail when you try to start the service.
discovery.type: single-node

# If you previously disabled the Security plugin in opensearch.yml,
# be sure to re-enable it. Otherwise you can skip this setting.
plugins.security.disabled: false

Сохраняем! 

sudo vim /opt/opensearch/config/jvm.options

Устанавлием сколько гигабайт отдадим opensearch. 

-Xms4g
-Xmx4g

Сохраняем! 

Обновляем переменные окружения 

export OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk

----------------------------------------------------------------------


Теперь выпустим TLS certificates

Так как у нас несколько нод мы будем выпускам скриптом. 

cd /opt/opensearch/config/

vim add_cert.sh

#!/bin/sh
# Не забываем менять DNS запись на актуальное hostname ваших нод! 
# Root CA
openssl genrsa -out root-ca-key.pem 2048
openssl req -new -x509 -sha256 -key root-ca-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=root.dns.a-record" -out root-ca.pem -days 730
# Admin cert
openssl genrsa -out admin-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in admin-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out admin-key.pem
openssl req -new -key admin-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=A" -out admin.csr
openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out admin.pem -days 730
# Node cert 1
openssl genrsa -out node1-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node1-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node1-key.pem
openssl req -new -key node1-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=node1.dns.a-record" -out node1.csr
echo 'subjectAltName=DNS:node1.dns.a-record' > node1.ext
openssl x509 -req -in node1.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node1.pem -days 730 -extfile node1.ext
# Node cert 2
openssl genrsa -out node2-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node2-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node2-key.pem
openssl req -new -key node2-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=node2.dns.a-record" -out node2.csr
echo 'subjectAltName=DNS:node2.dns.a-record' > node2.ext
openssl x509 -req -in node2.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out node2.pem -days 730 -extfile node2.ext

# так же со всеми нодами 

# Client cert Часто используется для opensearch-dashboard
openssl genrsa -out client-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in client-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out client-key.pem
openssl req -new -key client-key.pem -subj "/C=CA/ST=ONTARIO/L=TORONTO/O=ORG/OU=UNIT/CN=client.dns.a-record" -out client.csr
echo 'subjectAltName=DNS:client.dns.a-record' > client.ext
openssl x509 -req -in client.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out client.pem -days 730 -extfile client.ext
# Cleanup
rm admin-key-temp.pem
rm admin.csr
rm node1-key-temp.pem
rm node1.csr
rm node1.ext
rm node2-key-temp.pem
rm node2.csr
rm node2.ext
rm client-key-temp.pem
rm client.csr
rm client.ext


Сохраняем!
Делаем файл исполняемым.

sudo chomd  a+x  ./add_cert.sh

Запускам скрипт 

./add_cert.sh

Сертификаты должны лежать в /opt/opensearch/config/ на всех машинах свои!
Так же добавляем в запись в файл конфигурации 
# НЕ ЗАБЫВАЕМ ЗАКОМЕНТИРОВАТЬ/УДАЛИТЬ ссылки на демо конфигурации 

sudo vim /opt/opensearch/config/opensearch.yml

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
  - 'CN=node1.dns.a-record,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'
  - 'CN=node2.dns.a-record,OU=UNIT,O=ORG,L=TORONTO,ST=ONTARIO,C=CA'

# Не забываем менять ДНС записи на актуальные и названия ключей для кажной ноды нужны свои ключи кроме root и admin ключей! 

----------------------------

Настраиваем user который будет для входа в opensearch! 
Делайем икрипты исполняемым нашим пользователем

chmod 755 /opt/opensearch/plugins/opensearch-security/tools/*.sh

Переходим в директорию 

cd /opt/opensearch/plugins/opensearch-security/tools/

Запускам крипт 

./hash.sh

Если выдает ошибку то обновите переменнае окружения 

OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk ./hash.sh

Устанавливаем новый пароль и сохраняем Hash ключ 

Открываем файл 

sudo vim /opt/opensearch/config/opensearch-security/internal_users.yml

# Коментируем или удаляем всех пользоватлей кроме admin 
Пример:

---
# This is the internal user database
# The hash value is a bcrypt hash and can be generated with plugin/tools/hash.sh

_meta:
   type: "internalusers"
   config_version: 2

# Define your internal users here

admin:
   hash: "$2y$1EXAMPLEQqwS8TUcoEXAMPLEeZ3lEHvkEXAMPLERqjyh1icEXAMPLE." # Вставляем хеш который сформировали ранее!
   reserved: true
   backend_roles:
   - "admin"
   description: "Admin user"


Сохраняем!
--------------------------------------------------------------------------------


Применяем изменения 

Переходим в директорию 

cd /opt/opensearch/bin

Запускам сервис 

./opensearch

Переходим во второй терминал с подключением к данному серверу:

Переходим в директорию 

cd /opt/opensearch/plugins/opensearch-security/tools

Запускам иницилизацию плагина безопасности 

OPENSEARCH_JAVA_HOME=/opt/opensearch/jdk ./securityadmin.sh -cd /opt/opensearch/config/opensearch-security/ -cacert /opt/opensearch/config/root-ca.pem -cert /opt/opensearch/config/admin.pem -key /opt/opensearch/config/admin-key.pem -icl -nhnv

Сервер должен иницилизировать все сертификаты. 


---------------------------------------------------------------------------------


Создаем сервис 

sudo vi /etc/systemd/system/opensearch.service

[Unit]
Description=OpenSearch
Wants=network-online.target
After=network-online.target

[Service]
Type=forking
RuntimeDirectory=data

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



Сохраняем !

Перезугрузка демона 

sudo systemctl daemon-reload

Включаем автозазагрузки демона 

sudo systemctl enable opensearch.service

Включаем сервис 

sudo systemctl start opensearch

Проверяем статус сервиса 

sudo systemctl status opensearch

---------------------------------------------------------------------------


После того как мы выполнили данные операции на всех нодах которые мы хотим ввести в кластер. 
Останавливаем сервисы командой на всех нода  

sudo systemctl stop opensearch

После этого переходим на master-node-01 

sudo vim /opt/opensearch/config/opensearch.yml 

cluster.name: Имя вашего ноды
node.name: Ваше имя ноды 
node.roles: [ master , data ] 
network.host: [ адрес с которого будет доступин кластер или 0.0.0.0 ]
http.port: 9200 #порт по умолчани 
discovery.seed_hosts: [ "ip node ", "ip node" ]
cluster.initial_master_nodes: [ "dns name your master node ", "dns name your master node " ]
plugins.security.disabled: false

Важно что первая мастер нода должны запуститься с ролями  master и data.

После этого мы добавляем оставшиеся ноды с теми ролями которые они должны быть. 

Дата ноды указываем с ролей 

node.roles: [ data, ingest ]

Проверить что node добавились в кластер можно запросом к API 

curl -XGET https://<private-ip>:9200/_cat/nodes?v -u 'admin:<custom-admin-password>' --insecure

После добавляния всех нод можно с первой нодой убрать роль data и оставить только мастер! 

Важно что при добавлении нод может возникать ошибка что на машине присутсвует data 
Тогда следует удалить выполнить команду 

rm -rf /opt/opensearch/data/* 

Кластер собран осталось только установить Opensearch-dashboard что бы получить доступ дашборду! 