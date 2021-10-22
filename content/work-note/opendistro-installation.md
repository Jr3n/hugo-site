---
title: openDistro installation
date: 2019-12-31T16:40:09Z
draft: true
tags: []
series: 
categories:
toc: true
---
## 起因
ELK在近期越走越向商業發展，許多功能都是商業版才能使用。
AWS把他重新拿來再開發，並加上X-pack的部份功能！
像是 Alerting, Security, SQL, Performance Analyzer。

## Installation
### Demo 版
Demo版的以官方的文件直接進行安裝，基本上沒什麼問題。
[Docker compose file](https://opendistro.github.io/for-elasticsearch-docs/docs/install/docker/#sample-docker-compose-file)

### Production
ES的node之間的溝通需要證書，按照官網的文件產完了證書，做完後發現他在讀取證書的時候，會說證書缺少 `subjectAltName` 的 attribute。
不能動…算了吧，放生。

而此項功能是藉由 `Search Guard` 的module去完成的，於是就按照這個官網去產證書啦！
[Search Guard](https://search-guard.com/elasticsearch-tls-certificates-openssl/)

#### 準備 root CA 設定檔
首先，準備 root CA 的設定檔 `cert_config`：
```conf
[ req ]
prompt             = no
distinguished_name = root-ca

[ root-ca ]
commonName = root-ca
countryName = TW
organizationName = TECH
organizationalUnitName = IT
stateOrProvinceName = Taiwan
```

#### 準備 Node CA 設定檔
準備 node CA 的設定檔 `node_cert_config`：
```conf
[ req ]
prompt             = no
distinguished_name = node
req_extensions     = req_ext

[ node ]
commonName = odfe-node1
countryName = TW
organizationName = TECH
organizationalUnitName = IT
stateOrProvinceName = Taiwan

[ req_ext ]
subjectAltName = @alt_names
extendedKeyUsage = serverAuth,clientAuth

[alt_names]
DNS.0 = odfe-node1
IP.0 = 192.168.144.2
DNS.1 = odfe-node2
IP.1 = 192.168.144.3
```

#### 準備 Admin CA 設定檔
準備 admin CA 的設定檔 `admin_cert_config`：
```conf
[ req ]
prompt             = no
distinguished_name = node

[ node ]
commonName = ADMIN
countryName = TW
organizationName = TECH
organizationalUnitName = IT
stateOrProvinceName = Taiwan
```

#### 產生設定檔
```bash
# 產生 root CA
openssl req -x509 -newkey rsa:4096 -keyout root-ca-key.pem -out root-ca.pem -days 365 -nodes -config cert_config
# 產生 node CA
openssl req -newkey rsa:4096 -keyout node-key.pem -out node-cert.csr -days 365 -nodes -config node_cert_config
openssl x509 -req -in node-cert.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out node.pem -days 365 -extensions req_ext -extfile node_cert_config
# 產生 ADMIN CA
openssl req -newkey rsa:4096 -keyout admin-key.pem -out admin-cert.csr -days 365 -nodes -config admin_cert_config
openssl x509 -req -in admin-cert.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out admin.pem -days 365
# 刪除 csr
rm node-cert.csr
rm admin-cert.csr
```

#### Docker compose file
```yml
version: '3'
services:
  odfe-node1:
    image: amazon/opendistro-for-elasticsearch:1.9.0
    container_name: odfe-node1
    environment:
      - cluster.name=odfe-cluster
      - node.name=odfe-node1
      - discovery.seed_hosts=odfe-node1,odfe-node2
      - cluster.initial_master_nodes=odfe-node1,odfe-node2
      - bootstrap.memory_lock=true # along with the memlock settings below, disables swapping
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" # minimum and maximum Java heap size, recommend setting both to 50% of system RAM
      - network.host=0.0.0.0 # required if not using the demo security configuration
      - DISABLE_INSTALL_DEMO_CONFIG=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536 # maximum number of open files for the Elasticsearch user, set to at least 65536 on modern systems
        hard: 65536
    volumes:
      - odfe-data1:/usr/share/elasticsearch/data
      - ./root-ca.pem:/usr/share/elasticsearch/config/root-ca.pem
      - ./node.pem:/usr/share/elasticsearch/config/node.pem
      - ./node-key.pem:/usr/share/elasticsearch/config/node-key.pem
      - ./admin.pem:/usr/share/elasticsearch/config/admin.pem
      - ./admin-key.pem:/usr/share/elasticsearch/config/admin-key.pem
      - ./custom-elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    ports:
      - 9200:9200
      - 9600:9600 # required for Performance Analyzer
    networks:
      odfe-net:
        ipv4_address: 192.168.144.2

  odfe-node2:
    image: amazon/opendistro-for-elasticsearch:1.9.0
    container_name: odfe-node2
    environment:
      - cluster.name=odfe-cluster
      - node.name=odfe-node2
      - discovery.seed_hosts=odfe-node1,odfe-node2
      - cluster.initial_master_nodes=odfe-node1,odfe-node2
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - network.host=0.0.0.0
      - DISABLE_INSTALL_DEMO_CONFIG=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - odfe-data2:/usr/share/elasticsearch/data
      - ./root-ca.pem:/usr/share/elasticsearch/config/root-ca.pem
      - ./node.pem:/usr/share/elasticsearch/config/node.pem
      - ./node-key.pem:/usr/share/elasticsearch/config/node-key.pem
      - ./admin.pem:/usr/share/elasticsearch/config/admin.pem
      - ./admin-key.pem:/usr/share/elasticsearch/config/admin-key.pem
      - ./custom-elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    networks:
      odfe-net:
        ipv4_address: 192.168.144.3

  kibana:
    image: amazon/opendistro-for-elasticsearch-kibana:1.9.0
    container_name: odfe-kibana
    ports:
      - 5601:5601
    expose:
      - "5601"
    environment:
      ELASTICSEARCH_URL: https://odfe-node1:9200
      ELASTICSEARCH_HOSTS: https://odfe-node1:9200
    volumes:
      - ./custom-kibana.yml:/usr/share/kibana/config/kibana.yml
    networks:
      odfe-net:
        ipv4_address: 192.168.144.4

volumes:
  odfe-data1:
  odfe-data2:

networks:
  odfe-net:
    ipam:
      driver: default
      config:
        - subnet: "192.168.144.0/24"
```

#### custom-elasticsearch.yml
```yml
cluster.name: "odfe-cluster"
network.host: 0.0.0.0

# # minimum_master_nodes need to be explicitly set when bound on a public IP
# # set to 1 to allow single node clusters
# # Details: https://github.com/elastic/elasticsearch/pull/17288
# discovery.zen.minimum_master_nodes: 1

# # Breaking change in 7.0
# # https://www.elastic.co/guide/en/elasticsearch/reference/7.0/breaking-changes-7.0.html#breaking_70_discovery_changes
# cluster.initial_master_nodes:
#    - elasticsearch1
#    - docker-test-node-1

############## Open Distro Security configuration ###############

opendistro_security.ssl.transport.pemcert_filepath: node.pem
opendistro_security.ssl.transport.pemkey_filepath: node-key.pem
opendistro_security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
opendistro_security.ssl.transport.enforce_hostname_verification: true
opendistro_security.ssl.http.enabled: true
opendistro_security.ssl.http.pemcert_filepath: node.pem
opendistro_security.ssl.http.pemkey_filepath: node-key.pem
opendistro_security.ssl.http.pemtrustedcas_filepath: root-ca.pem
opendistro_security.allow_unsafe_democertificates: false

############## Common configuration settings ##############

# Enable or disable the Open Distro Security advanced modules
# By default advanced modules are enabled, you can switch
# all advanced features off by setting the following key to false
opendistro_security.advanced_modules_enabled: true

# Specify a list of DNs which denote the other nodes in the cluster.
# This settings support wildcards and regular expressions
# The list of DNs are also read from security index **in addition** to the yml configuration if
# opendistro_security.nodes_dn_dynamic_config_enabled is true.
# NOTE: This setting only has effect if 'opendistro_security.cert.intercluster_request_evaluator_class' is not set.
opendistro_security.nodes_dn_dynamic_config_enabled: false
opendistro_security.nodes_dn:
  - 'ST=Taiwan,OU=IT,O=WCS,C=TW,CN=odfe-node1'

# Defines the DNs (distinguished names) of certificates
# to which admin privileges should be assigned (mandatory)
opendistro_security.authcz.admin_dn:
  - 'ST=Taiwan,OU=IT,O=WCS,C=TW,CN=ADMIN'
# Define how backend roles should be mapped to Open Distro Security roles
# MAPPING_ONLY - mappings must be configured explicitely in roles_mapping.yml (default)
# BACKENDROLES_ONLY - backend roles are mapped to Open Distro Security rules directly. Settings in roles_mapping.yml have no effect.
# BOTH - backend roles are mapped to Open Distro Security roles mapped directly and via roles_mapping.yml in addition
opendistro_security.roles_mapping_resolution: MAPPING_ONLY

opendistro_security.audit.type: internal_elasticsearch
opendistro_security.enable_snapshot_restore_privilege: true
opendistro_security.check_snapshot_restore_write_privileges: true
opendistro_security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
cluster.routing.allocation.disk.threshold_enabled: false
opendistro_security.audit.config.disabled_rest_categories: NONE
opendistro_security.audit.config.disabled_transport_categories: NONE
```

#### custom-kibana.yml
```yml
---
# Copyright 2019 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License").
# You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# or in the "license" file accompanying this file. This file is distributed
# on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either
# express or implied. See the License for the specific language governing
# permissions and limitations under the License.

# Description:
# Default Kibana configuration from kibana-docker.

server.port: 5601

server.name: kibana
server.host: 0.0.0.0

elasticsearch.hosts: https://odfe-node1:9200
elasticsearch.ssl.verificationMode: none
elasticsearch.username: kibanaserver
elasticsearch.password: kibanaserver
elasticsearch.requestHeadersWhitelist: ["securitytenant","Authorization"]

server.ssl.enabled: false

opendistro_security.multitenancy.enabled: true
opendistro_security.multitenancy.tenants.preferred: ["Private", "Global"]
opendistro_security.readonly_mode.roles: ["kibana_read_only"]

newsfeed.enabled: false
telemetry.optIn: false
telemetry.enabled: false
```

#### 組裝啦！
```bash
docker-compose up -d
# 接下來進入 odfe-node1 設定安全性
cd plugins/opendistro_security/tools/
sh ./securityadmin.sh -cd ../securityconfig/ -icl -nhnv -cacert ../../../config/root-ca.pem -cert ../../../config/admin.pem -key ../../../config/admin-key.pem
```

如果可以很順利的通過的話，恭喜你剛完成…最初的一步…
可以打上 `https://<server-ip>:9200/` 去驗證不是能夠運作
也可以打上 `http://<server-ip>:5601/` 看一下 kibana 是不是有正常運行