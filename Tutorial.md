# 🚀 Arquitetura de Sincronização em Tempo Real (CDC)
**Origem:** SQL Server ➡️ **Mensageria:** Kafka (Debezium) ➡️ **Destino:** PostgreSQL

Este repositório contém a infraestrutura e os scripts necessários para levantar um ambiente local de *Change Data Capture* (CDC). O fluxo captura alterações no SQL Server em tempo real e as insere automaticamente no PostgreSQL usando o Kafka Connect.

---

## 📋 Pré-requisitos
* **Docker Desktop** instalado.
* **Memória RAM:** Acesse as configurações do Docker e garanta que ele tem permissão para usar **pelo menos 8GB de RAM**. O ecossistema Java (Kafka) e o SQL Server exigem bastante memória.
* Cliente SQL (DBeaver, Azure Data Studio, etc.) para conectar aos bancos.

---

## 🛠️ Passo 1: Subindo a Infraestrutura (Docker)

Na raiz do projeto, crie o arquivo `docker-compose.yml` com a estrutura abaixo. Este arquivo já baixa as dependências corretas e contorna problemas comuns de repositório.

```yaml
version: '3.8'

services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: mssql_primary
    ports:
      - "1433:1433"
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "Password!123"
      MSSQL_PID: "Developer"
      MSSQL_AGENT_ENABLED: "true" # Obrigatório para o CDC

  postgres:
    image: postgres:15-alpine
    container_name: postgres_replica
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: Password!123
      POSTGRES_DB: replica_db

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    container_name: kafka_connect
    depends_on:
      - kafka
      - sqlserver
      - postgres
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:29092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: "compose-connect-group"
      CONNECT_CONFIG_STORAGE_TOPIC: "my_connect_configs"
      CONNECT_OFFSET_STORAGE_TOPIC: "my_connect_offsets"
      CONNECT_STATUS_STORAGE_TOPIC: "my_connect_statuses"
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_REST_ADVERTISED_HOST_NAME: "kafka_connect"
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
    command:
      - bash
      - -c
      - |
        echo "Instalando JDBC Sink Connector..."
        confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:latest
        
        echo "Baixando Debezium SQL Server direto da fonte oficial..."
        mkdir -p /usr/share/confluent-hub-components/debezium-sqlserver
        curl -L https://repo1.maven.org/maven2/io/debezium/debezium-connector-sqlserver/2.5.0.Final/debezium-connector-sqlserver-2.5.0.Final-plugin.tar.gz | tar -xz -C /usr/share/confluent-hub-components/debezium-sqlserver --strip-components=1
        
        echo "Iniciando Kafka Connect..."
        /etc/confluent/docker/run
```

Execute no terminal:
```bash
docker-compose up -d
```
Aguarde até que o log do Connect (`docker logs -f kafka_connect`) exiba `INFO Kafka Connect started`.

---

## 🗄️ Passo 2: Configurando a Origem (SQL Server)

Conecte-se ao SQL Server (`localhost:1433`, SA / `Password!123`) e execute:

```sql
CREATE DATABASE loja_db;
USE loja_db;

CREATE TABLE cliente (
	id bigint IDENTITY(1,1) NOT NULL PRIMARY KEY,
	nome varchar(100) NOT NULL,
	datanascimento datetime NOT NULL,
	documento varchar(11) NOT NULL,
	datacadastro datetime NOT NULL,
	dataalteracao datetime NULL,
	situacao bit DEFAULT 0 NOT NULL
);

-- Ativa o CDC no Banco e na Tabela
EXEC sys.sp_cdc_enable_db;
EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name   = N'cliente',
    @role_name     = NULL;
```

---

## 🐘 Passo 3: Configurando o Destino (PostgreSQL)

Conecte-se ao PostgreSQL (`localhost:5432`, banco `replica_db`, admin / `Password!123`) e execute:

```sql
CREATE TABLE cliente (
    id BIGINT PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    datanascimento TIMESTAMP NOT NULL,
    documento VARCHAR(11) NOT NULL,
    datacadastro TIMESTAMP NOT NULL,
    dataalteracao TIMESTAMP NULL,
    situacao BOOLEAN DEFAULT FALSE NOT NULL
);

-- Cast Implícito: Ensina o Postgres a converter os milissegundos do Kafka em Timestamp
CREATE OR REPLACE FUNCTION convert_bigint_to_timestamp(milis bigint)
RETURNS timestamp without time zone AS $$
    SELECT to_timestamp(milis / 1000.0)::timestamp without time zone;
$$ LANGUAGE sql IMMUTABLE;

CREATE CAST (bigint AS timestamp without time zone)
WITH FUNCTION convert_bigint_to_timestamp(bigint) AS IMPLICIT;
```

---

## 🌉 Passo 4: Ativando a Ponte (Kafka Connect)

Crie os dois arquivos JSON abaixo na raiz do projeto.

**Arquivo `sqlserver.json` (Leitura do SQL):**
```json
{
  "name": "sqlserver-source",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "tasks.max": "1",
    "database.hostname": "mssql_primary",
    "database.port": "1433",
    "database.user": "sa",
    "database.password": "Password!123",
    "database.names": "loja_db",
    "topic.prefix": "app",
    "table.include.list": "dbo.cliente",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:29092",
    "schema.history.internal.kafka.topic": "schema-changes.loja_db",
    "database.encrypt": "false",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "true",
    "value.converter.schemas.enable": "true"
  }
}
```

**Arquivo `postgres.json` (Escrita no Postgres):**
```json
{
  "name": "postgres-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "app.loja_db.dbo.cliente",
    "connection.url": "jdbc:postgresql://postgres_replica:5432/replica_db",
    "connection.user": "admin",
    "connection.password": "Password!123",
    "insert.mode": "upsert",
    "pk.mode": "record_value",
    "pk.fields": "id",
    "auto.create": "false",
    "auto.evolve": "false",
    "transforms": "unwrap,route",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "app\\.loja_db\\.dbo\\.(.*)",
    "transforms.route.replacement": "$1",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "true",
    "value.converter.schemas.enable": "true"
  }
}
```

Dispare os comandos via terminal para iniciar os conectores:
```bash
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d @sqlserver.json
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d @postgres.json
```

---

## 🧪 Passo 5: Testando o Fluxo

1. Insira um dado no **SQL Server**:
```sql
USE loja_db;
INSERT INTO cliente (nome, datanascimento, documento, datacadastro, situacao)
VALUES ('Engenheiro de Dados', '1995-01-01', '12345678901', GETDATE(), 1);
```

2. Consulte imediatamente no **PostgreSQL**:
```sql
SELECT * FROM cliente;
```

---
💡 **Dica de Troubleshooting (Reset Limpo):**
Se precisar limpar o ambiente inteiro por causa de lixo de memória ou cache do Zookeeper, rode `docker-compose down -v`.