# CUSTOMER360
C4 MODEL - CUSTOMER 360 / PONTO ÚNICO DE CONSULTA

Resumo da estratégia (alto nível).
Diagrama C4 (Context → Containers → Componentes).
Fluxo de dados e tecnologias escolhidas (por que e como usar).
Regras de consolidação / resolução de conflito.
Como garantir latência < 1ms e demais SLAs.
Resiliência e tolerância a falhas (legados/infra).
Mapeamento para OpenAPIs (TM Forum) com justificativa e links.


Diagrama (arte detalhado)
[Salesforce] --(platform events or REST)-> [Integration Adapter] \
                                                            \
[Legacy Oracle] --(CDC / Debezium/GoldenGate)-> [Kafka Topics] ---> [Stream Processor (ksql/Streams/Flink)] -> [MDM Service] -> [Event: customer360.updated] -> [Updater] -> Redis (hot) + Elastic (index)
                                                                                                                                                                                   
[CustomerQuery API (Spring Boot)] <---- API Gateway (Kong/Apigee) <---- [Portal / Salesforce UI / MeuVivo]


### 1) Estratégia resumida (one-liner)

- Criar um **Data Hub read-store** (denormalized, in-memory / search store) que será a **fonte única de consulta** (Customer Query API).
- Esse hub é alimentado por **eventos (CDC / mensagens)** vindos dos legados e do Salesforce.
- Os dados passam por um pipeline de **enriquecimento / MDM / regras de precedência**.
- O hub persiste **vistas prontas** em Redis / Elastic para leitura **ultra-rápida**.
- A API **nunca consulta sistemas legados online**.

---

### O Desafio & Visão da Solução

- Garantir a consulta do **"Melhor Dado"** do cliente em um único ponto (API).
- Atingir tempo de resposta **abaixo de milissegundos**.
- Superar a fragmentação e coexistência de dados entre Sistemas de Registro (SORs), legados e Salesforce.
- A solução proposta é a implementação de um **Cliente Data Hub (CDH)** baseado em:
  - Arquitetura Orientada a Eventos (EDA)
  - Segregação de responsabilidades de leitura e escrita (CQRS-like)

---

### 2) Estratégia de Consolidação e Fluxo de Dados

#### A. O Conceito de Golden Record

- **Definição de Regras**
  - Primeiro passo: definir a lógica de Golden Record.
  - Determina qual sistema é a **Fonte de Verdade (System of Record - SOR)** para cada atributo do cliente.

- **Modelo Canônico**
  - Criação de um modelo de dados **padrão, único e centralizado**.
  - O CDH usa esse modelo como estrutura-base para armazenar o Golden Record.

---

#### B. Fluxo de Atualização (Write — Assíncrono)

O fluxo deve garantir que o sistema de consulta **não dependa diretamente** dos sistemas fontes.

- **Alteração**
  - O dado é alterado no **CRM (Salesforce)** ou em um **Sistema Legado**.

- **Geração de Evento**
  - **CRM:** Publica um evento padronizado (`CustomerDataChanged`) no Broker de Mensagens.
  - **Legado:**  
    - Uso de **Change Data Capture (CDC)** (ex.: Debezium).  
    - Um **Microsserviço Adaptador (ACL)** monitora o banco legado.  
    - Tradução do dado para o **modelo canônico**.  
    - Publicação do evento no Broker.

- **Consolidação**
  - O **Customer Sync Worker** (microsserviço):
    - Consome o evento.
    - Aplica a lógica de **Golden Record**.
    - Atualiza o **Customer360 DB** (fonte de verdade consolidada).



### 3) Estratégia de Performance e Tecnologias

Para atingir o requisito de **resposta sub-milissegundos** e **alta assertividade**, a arquitetura de consulta é **isolada e otimizada**:

A. Tecnologias

## Tecnologias — Camadas e Benefícios (MDM / Customer 360)

| **Camada**              | **Tecnologia**                 | **Benefício** |
|-------------------------|--------------------------------|---------------|
| Integração/Eventos      | Apache Kafka                   | Garante alta vazão, durabilidade e ordem dos eventos (crucial para MDM). |
| Consulta Rápida         | Redis Cache                    | Implementa a camada Read-Store. O ponto de consulta mais rápido (Cache-Aside). |
| SSOT                    | PostgreSQL (JSONB)             | Armazena o Golden Record, suportando consultas otimizadas e dados flexíveis. |
| Microsserviços          | Spring Boot (Java/Quarkus)     | Performance e resiliência para a Customer Query API e o Sync Worker. |

### B. Otimização da Consulta (Customer Query API)

- **Prioridade do Cache**
  - A Customer Query API consulta primeiramente o Redis Cache.
  - Se o dado estiver lá (**Cache Hit**), a resposta é imediata (< 1ms).

- **Atualização do Cache**
  - O *Customer Sync Worker* é o único responsável por invalidar ou atualizar a chave do cliente no Redis após a escrita no banco principal.
  - Isso garante a **frescura e consistência** do dado.

- **APIficação**
  - Uso do padrão **OpenAPI TMF632 (Customer Management)** para a API de consulta.
  - Garante aderência a padrões de Telecom e interoperabilidade entre sistemas.

  ### 4) Estratégia de Resiliência e Robustez

A resiliência é fundamental, especialmente ao lidar com sistemas legados:

- **Persistência de Eventos (Kafka)**
  - Se o *Customer Sync Worker* falhar, o Kafka retém os eventos de alteração.
  - Quando o Worker volta, ele continua o processamento de onde parou.
  - Isso garante **Consistência Eventual**, mesmo em cenários de falha temporária.

- **Dead Letter Queue (DLQ)**
  - Em caso de falha permanente no processamento de uma mensagem (ex.: erro de formato), ela é enviada para uma DLQ no Kafka.
  - Isso evita que uma mensagem "venenosa" (*poison pill*) paralise o tópico inteiro.

- **Circuit Breaker**
  - Implementado na **Customer Query API**.
  - Se o Redis ou o Customer360 DB apresentar falhas ou alta latência, o Circuit Breaker é acionado.
  - Isso protege o sistema de consulta e permite retornar uma resposta de **fallback** (ex.: dado *stale*).
  - Mantém a **disponibilidade do serviço** mesmo em períodos de instabilidade.







