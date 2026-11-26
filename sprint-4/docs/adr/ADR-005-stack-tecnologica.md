# ADR-005: Seleção de Frameworks, Banco Vetorial e Stack do API Gateway

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-11-05 |
| Decisores | Equipe completa |
| Cenários relacionados | C1, C3, C4, C5 (QAW) |

## Contexto

Com estilo e protocolos definidos (ADR-001 e ADR-003), falta escolher a stack concreta. Critérios, em ordem de peso: desempenho no laço do jogo, domínio da equipe, maturidade, custo operacional.

## Decisões

### D1. Backend: .NET 10 (LTS) com ASP.NET Core

| Alternativa | Avaliação |
|---|---|
| **.NET 10 (escolhida)** | Alto desempenho em WebSockets e gRPC, `Span<T>` e structs para evitar alocações no laço, Server GC, LTS até novembro de 2028. Ecossistema pedido pela disciplina (Polly, YARP, ArchUnitNET). |
| Node.js | Uma linguagem com o cliente, mas laço em thread única e GC menos controlável. |
| Go | Ótimo para servidores de rede, fora do domínio da equipe. |
| Rust | Melhor desempenho, curva de aprendizado incompatível com o prazo. |

### D2. Cliente: TypeScript + Three.js + Vite

| Alternativa | Avaliação |
|---|---|
| **Three.js (escolhida)** | Leve, controle total do laço de renderização (necessário para predição e interpolação). |
| Babylon.js | Mais completa, mais pesada, impõe sua estrutura. |
| Unity WebGL | Build de dezenas de MB, carregamento lento; contraria "abrir o link e jogar". |

### D3. API Gateway: YARP

| Critério | YARP | Ocelot | Kong / Envoy |
|---|---|---|---|
| Proxy de WebSocket | Sim | Limitado | Sim |
| Afinidade por instância (sticky) | Sim, configurável por código | Básica | Sim |
| Extensão em C# (middleware ASP.NET) | Nativa | Nativa | Plugins em Lua / filtros em C++ |
| Manutenção | Microsoft, ativa | Comunidade, ritmo menor | Ativa |
| Operação | Mais um app .NET | Mais um app .NET | Componente separado |

**Escolhido: YARP.** WebSocket com afinidade (rotear o jogador à instância da partida dele), mesmo pipeline do ASP.NET Core para JWT e rate limiting, sem tecnologia nova para operar.

### D4. Event streaming: Apache Kafka (KRaft) + Schema Registry

Escolha do ADR-003. Cliente .NET: `Confluent.Kafka`. Contratos em Avro versionados no Schema Registry, com regra de compatibilidade retroativa (`BACKWARD`) para que consumidores antigos não quebrem quando um evento ganha campos.

### D5. Banco vetorial: pgvector

| Alternativa | Avaliação |
|---|---|
| **pgvector (escolhida)** | Extensão do PostgreSQL que já usamos. Vetores de 12 dimensões e dezenas de milhares de casos revisados: muito abaixo do limite. Mesma transação do caso revisado. |
| Qdrant | Excelente para milhões de vetores com filtros, mas é mais um banco para operar sem necessidade nesta escala. |
| Pinecone | Gerenciado, mas custo recorrente e dados fora da infraestrutura. |

Revisitar se o RAG passar a usar embeddings de texto (centenas de dimensões) em milhões de documentos.

### D6. Demais componentes

| Necessidade | Escolha | Motivo |
|---|---|---|
| Resiliência e Bulkhead | Polly v8 (`Microsoft.Extensions.Http.Resilience`) | Circuit Breaker, timeout e limitador de concorrência por compartimento |
| Cache e fila de matchmaking | Redis | Sorted sets para faixa de Elo, contadores de coocorrência |
| Persistência | PostgreSQL 17 + EF Core | Schemas por módulo; banco separado para pagamentos |
| Identidade | Keycloak (OIDC) | Subdomínio genérico |
| Pagamentos | PSP PIX e adquirente de cartão via API, cartão tokenizado no navegador | Escopo PCI mínimo (SAQ A) |
| SLM local | Ollama com modelo de 3 a 8B quantizado | Tier 1 sem custo por chamada |
| LLM em nuvem | Claude Opus 5.5 via SDK oficial da Anthropic para C# | Structured outputs com JSON Schema, effort ajustável |
| Orquestração | Kubernetes + Agones | Autoscaling de Game Servers com buffer aquecido, sem derrubar partidas em andamento (C4) |
| Observabilidade | OpenTelemetry + Prometheus + Grafana | Métricas de tick, lag de consumidores Kafka, estado dos circuitos |
| Testes de arquitetura | ArchUnitNET + xUnit | Fitness functions |
| Testes de carga | k6 + enxame de bots em Node.js | C1, C2 |

## Consequências

**Positivas**
- Uma linguagem no backend inteiro; o gateway reaproveita o conhecimento de ASP.NET Core.
- Nenhum banco extra para busca vetorial.
- Agones resolve o problema específico de escalar servidores de jogo com estado.

**Negativas**
- Física duplicada em C# (servidor) e TypeScript (predição); exige testes de paridade.
- Kafka, Kubernetes e Agones são complexos para o ambiente acadêmico; a demonstração usa Docker Compose (Kafka em modo KRaft de nó único), com o desenho de produção documentado.
