Generate API KEYS for the Kafka cluster and Schema Registry

<details>
<summary>API Keys</summary>


Kafka
```
TWXYG6XASDQR4OIX:cfltgiNucYDmJux5sdfsdfNh2IRsdfsdfWSQc/hHPEzu29SuEkzy7TTKTi3c5sxQ
```

Schema
```
M6JERTU2XWPNBEYA:cflt2dTvCWov6vOAjD6P2VsdfsdfsdfbdDMjDGT/bJN3zQnfuFlDxLWDq/w
```

</details>

## Ubuntu 24.04 server setup

### Setup software:
```bash
sudo mkdir -p /etc/apt/keyrings

wget -qO - https://packages.confluent.io/deb/8.2/archive.key | gpg \
--dearmor | sudo tee /etc/apt/keyrings/confluent.gpg > /dev/null
```

```bash
CP_DIST=$(lsb_release -cs)
echo "Types: deb
URIs: https://packages.confluent.io/deb/8.2
Suites: stable
Components: main
Architectures: $(dpkg --print-architecture)
Signed-by: /etc/apt/keyrings/confluent.gpg

Types: deb
URIs: https://packages.confluent.io/clients/deb/
Suites: ${CP_DIST}
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/confluent.gpg" | sudo tee /etc/apt/sources.list.d/confluent-platform.sources > /dev/null
```

```bash
sudo apt-get update && sudo apt-get install confluent-server confluent-schema-registry openjdk-21-jdk plocate unzip -y
sudo updatedb
```

To test the setup later:
```bash
tee local-file-source.json <<EOF
{
  "name": "local-file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/tmp/a.txt",
    "topic": "fromubun"
  }
}
EOF
```

## Amazon Linux 2023 server setup

### Setup software:
```bash
sudo rpm --import https://packages.confluent.io/rpm/8.2/archive.key

sudo tee /etc/yum.repos.d/confluent.repo <<EOF
[Confluent]
name=Confluent repository
baseurl=https://packages.confluent.io/rpm/8.2
gpgcheck=1
gpgkey=https://packages.confluent.io/rpm/8.2/archive.key
enabled=1

[Confluent-Clients]
name=Confluent Clients repository
baseurl=https://packages.confluent.io/clients/rpm/centos/9/x86_64
gpgcheck=1
gpgkey=https://packages.confluent.io/clients/rpm/archive.key
enabled=1
EOF
```

```bash
sudo yum clean all && sudo yum install confluent-server confluent-schema-registry java-21-amazon-corretto-devel.x86_64 mlocate unzip -y
sudo updatedb
```
To test the setup later:
```bash
tee local-file-source.json <<EOF
{
  "name": "local-file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/tmp/a.txt",
    "topic": "fromamzn"
  }
}
EOF
```

## Both servers

### Get and install connectors
```bash
wget https://hub-downloads.confluent.io/api/plugins/debezium/debezium-connector-sqlserver/versions/3.2.6/debezium-debezium-connector-sqlserver-3.2.6.zip

wget https://hub-downloads.confluent.io/api/plugins/confluentinc/kafka-connect-jdbc/versions/10.9.2/confluentinc-kafka-connect-jdbc-10.9.2.zip

wget https://hub-downloads.confluent.io/api/plugins/snowflakeinc/snowflake-kafka-connector/versions/3.5.3/snowflakeinc-snowflake-kafka-connector-3.5.3.zip

wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-jdbc/3.2.6.Final/debezium-connector-jdbc-3.2.6.Final-plugin.zip

sudo mkdir -p /opt/connectors && sudo chown cp-kafka-connect:confluent /opt/connectors
sudo unzip confluentinc-kafka-connect-jdbc-10.9.2.zip -d /opt/connectors/
sudo unzip snowflakeinc-snowflake-kafka-connector-3.5.3.zip -d /opt/connectors/
sudo unzip debezium-debezium-connector-sqlserver-3.2.6.zip -d /opt/connectors/
sudo unzip debezium-connector-jdbc-3.2.6.Final-plugin.zip -d /opt/connectors/
sudo cp /opt/connectors/debezium-debezium-connector-sqlserver-3.2.6/lib/mssql-jdbc-12.4.2.jre8.jar /opt/connectors/
sudo chown -R cp-kafka-connect:confluent /opt/connectors/debezium-connector-jdbc
sudo chmod 775 /var/log/kafka
```

### Configure connect
```bash
sudo mv /etc/kafka/connect-distributed.properties /etc/kafka/connect-distributed.properties_keep
```

Create `/etc/kafka/connect-distributed.properties` with content locate in [connect-distributed.properties](connect-distributed.properties).

Values that need to be replaced:
- `<cloud bootstrap servers>` - bootstrap servers of the cloud cluster
- `<connect cluster id>` - if there are more than one connect clusters, give each one of them id
- `<API KEY>` - API KEY generated in the cloud
- `<API SECRET>` - Secret that is generated with the key

### Start Kafka Connect

Enable and start the service:

```bash
sudo systemctl enable --now confluent-kafka-connect.service
```

Monitor the startup process:

```bash
sudo tail -f /var/log/kafka/connect.log
```

### Check connectors

```bash
curl -s http://localhost:8083/connector-plugins | jq
```

Expected output:
```json
[
  {
    "class": "com.snowflake.kafka.connector.SnowflakeSinkConnector",
    "type": "sink",
    "version": "3.5.3"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "type": "sink",
    "version": "10.9.2"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "type": "source",
    "version": "10.9.2"
  },
  {
    "class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "type": "source",
    "version": "3.2.6.Final"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "8.2.0-ce"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "8.2.0-ce"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "8.2.0-ce"
  }
]

```

### Test connectors connectivity

Create topics `fromamzn` and `fromubun` in the cloud cluster.

Run (on both servers):
```bash
echo '{"text" : "hello"}' > /tmp/a.txt
curl -s -H "Content-Type: application/json" -X POST -d @local-file-source.json http://localhost:8083/connectors/ | jq .
```

Check that connector was created and running (might take a minute or two):
```bash
curl -s http://localhost:8083/connectors | jq
curl -s http://localhost:8083/connectors/local-file-source/status | jq
```

Expected output:
```json
{
  "name": "local-file-source",
  "connector": {
    "state": "RUNNING",
    "worker_id": "10.0.93.174:8083",
    "version": "8.2.0-ce"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "10.0.93.174:8083",
      "version": "8.2.0-ce"
    }
  ],
  "type": "source"
}
```

Check contents of both topics `fromamzn` and `fromubun`, expected output:
<img width="899" height="462" alt="image" src="https://github.com/user-attachments/assets/5f031589-c814-4ca1-8033-ae0ddd332921" />

Once everything is running, remove the connector and topics:
```bash
curl -X DELETE http://localhost:8083/connectors/local-file-source
```

### Communication with CC from CLI

```bash
sudo curl -sL --http1.1 https://cnfl.io/cli | sudo sh -s -- -b /usr/local/bin
confluent login --save --prompt
confluent kafka cluster list
confluent kafka cluster use lkc-w95dn9
confluent api-key create --resource lkc-w95dn9 --description "My-CLI-Key"
confluent api-key use YBLAWZYGPPG43GUO --resource lkc-w95dn9
confluent kafka topic list
confluent kafka topic consume sqlserver.crm.dbo.Customers --from-beginning
```

### Topics to create for MSSQL Debezium

Database is `crm`, schema is `dbo`, table is `Customers`:

- schemahistory.crm
- sqlserver
- sqlserver.crm.dbo.Customers

Partition count `1`


































