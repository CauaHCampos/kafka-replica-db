# Kafka Data Replication Pipeline (CDC)

Este repositório contém a implementação de uma arquitetura de **Change Data Capture (CDC)** para replicação de dados em tempo real. O foco principal é garantir a consistência, a resiliência e a alta disponibilidade dos dados trafegados entre sistemas de origem e destino.

---

## 🏗️ Arquitetura do Sistema

O pipeline foi desenhado utilizando o ecossistema **Apache Kafka** para atuar como um barramento de eventos distribuído, permitindo a sincronização de bases de dados com baixíssima latência.

### Componentes Principais
* **Apache Kafka:** Cluster de mensageria para persistência e distribuição dos eventos.
* **Debezium:** Conector de captura de dados (CDC) que monitora os logs de transação (WAL/Binlog).
* **Kafka Connect:** Framework para execução e gerenciamento dos conectores de Source e Sink.
* **Zookeeper:** Coordenação do estado do cluster e eleição de líderes de partição.
* **Docker & Docker Compose:** Orquestração de containers para ambiente de desenvolvimento e teste.

---

## 🛡️ Estratégias de Resiliência

Para assegurar a integridade da replicação e evitar a perda de dados, o projeto aplica as seguintes diretrizes técnicas:

* **Replication Factor:** Configurado para garantir que cada partição possua réplicas em múltiplos brokers.
* **In-Sync Replicas (ISR):** Garantia de que as mensagens sejam confirmadas apenas após a sincronização entre o Leader e os Followers.
* **Acks (Acknowledgements):** Utilização de `acks=all` para máxima segurança na escrita dos eventos.
* **Offset Management:** Controle rigoroso da posição de leitura para garantir que nenhum evento seja perdido em caso de restart dos serviços.

