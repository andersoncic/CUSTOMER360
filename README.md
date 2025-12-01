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


