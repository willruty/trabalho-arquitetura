# Sprint 3: Integração e Resiliência

**Período:** 07/10/2026 a 04/11/2026 | **Marcos:** ADR-003 (20/10/2026) e ADR-004 (28/10/2026)

## Objetivo da sprint

Definir como as partes do sistema conversam entre si e com o mundo externo: o canal bidirecional com os jogadores, a mensageria assíncrona com Apache Kafka, o mapa de contexto DDD com Camadas Anti-Corrupção e a **estratégia de Bulkhead para isolar o gateway de pagamentos PIX/Cartão** (cenários C1, C3, C4, C6 e C7).

## Equipe e funções nesta sprint

| Integrante | Função na Sprint 3 | Responsabilidade principal |
|---|---|---|
| [Nome do Aluno 3] | Liderança | Resiliência, Bulkhead de pagamentos e integração com o legado |
| [Nome do Aluno 2] | Pesquisador | Mapa de contexto e ACLs (ADR-004) |
| [Nome do Aluno 1] | Pesquisador | Netcode: predição, interpolação, compensação de latência |
| [Nome do Aluno 4] | Executor | Canal bidirecional, tópicos Kafka e ADR-003 |
| [Nome do Aluno 5] | Executor | C4 nível 2 (v2) e diagramas de sequência |

## Entregáveis

| # | Entregável | Documento |
|---|---|---|
| 1 | Canal bidirecional, protocolos, tópicos Kafka e netcode | [docs/05-protocolos-e-netcode.md](docs/05-protocolos-e-netcode.md) |
| 2 | Mapa de contexto DDD e Camadas Anti-Corrupção | [docs/06-mapa-de-contexto-ddd.md](docs/06-mapa-de-contexto-ddd.md) |
| 3 | Resiliência e estratégia de Bulkhead PIX/Cartão | [docs/07-resiliencia.md](docs/07-resiliencia.md) |
| 4 | C4 Nível 2: Contêineres (versão 2) | [docs/c4/c2-containers-v2.md](docs/c4/c2-containers-v2.md) |
| 5 | ADR-003: Mensageria assíncrona e protocolos de integração | [docs/adr/ADR-003-protocolos-de-integracao.md](docs/adr/ADR-003-protocolos-de-integracao.md) |
| 6 | ADR-004: Contextos delimitados e ACL | [docs/adr/ADR-004-contextos-e-acl.md](docs/adr/ADR-004-contextos-e-acl.md) |

## Critérios de aceite da sprint

- [x] C4 nível 2 evidenciando o canal bidirecional (WebSocket e gRPC streaming) e o Apache Kafka.
- [x] Tópicos Kafka com chave de partição justificada (ordem por partida).
- [x] Bulkhead do gateway PIX/Cartão com compartimentos e limites numéricos.
- [x] Mapa de contexto com o tipo de relacionamento entre cada par de contextos.
- [x] ACL com Adapter isolando o modelo XML do legado e os modelos dos provedores de pagamento.
- [x] Fluxo completo de inscrição paga em torneio num diagrama de sequência.

## Retrospectiva

**O que funcionou:** tratar cada sistema externo como "inimigo" (a Federação vai cair, a adquirente vai demorar, o PSP vai mandar webhook duplicado) levou a um desenho em que nenhuma falha externa chega perto de uma partida.

**O que melhorar:** a escolha entre RabbitMQ e Kafka gerou debate longo. O critério decisivo foi o fluxo de eventos de combate em tempo real para a IA (~25.000 eventos/s, com ordem por partida e necessidade de reprocessamento), registrado no ADR-003.

## Backlog levado para a Sprint 4

- Agente de IA de trapaça e conluio sobre o fluxo Kafka: model tiering, RAG e guardrails (ADR-006).
- Stack final, banco vetorial e API Gateway (ADR-005).
- Fitness functions (ArchUnitNET) e métricas de instabilidade e abstração.
- Documento final consolidado.
