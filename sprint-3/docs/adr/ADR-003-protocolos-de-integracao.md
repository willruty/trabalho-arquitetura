# ADR-003: Mensageria Assíncrona e Protocolos de Integração (WebSocket, gRPC, REST e Kafka)

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-10-20 |
| Decisores | Equipe completa |
| Cenários relacionados | C1, C2, C3, C4, C5 (QAW) |

## Contexto

Com três artefatos de deploy (ADR-001), há cinco tipos de comunicação:

1. **Jogador e Game Server:** 30 mensagens/s em cada direção, latência crítica.
2. **Árbitro e partida de torneio:** estado ao vivo num sentido, comandos (pausar) no outro.
3. **Matchmaking e Game Server:** pedido de alocação com resposta imediata.
4. **Game Server e plataforma:** fluxo contínuo de eventos de combate para o agente de IA (~25.000 eventos/s no pico), mais o fim de cada partida. O Game Server não pode esperar ninguém.
5. **Payment Service e plataforma:** confirmação de pagamentos, que não pode se perder.

## Decisões

### D1. Jogador e jogo: WebSocket com MessagePack (canal bidirecional)

| Alternativa | Avaliação |
|---|---|
| WebSocket + JSON | Simples de depurar, ~2,5x maior que binário. |
| **WebSocket + MessagePack (escolhida)** | Binário compacto, suportado no navegador e no .NET; JSON fica como modo de depuração. |
| WebRTC DataChannel (UDP) | Evita bloqueio de fila do TCP, mas exige STUN/TURN e biblioteca WebRTC no servidor. Evolução futura. |
| gRPC-Web | Não suporta streaming bidirecional no navegador. |

### D2. Árbitro e partida: gRPC bidirectional streaming

Um stream por partida de torneio entre Platform API e Game Server (`MatchControl.Control`): o Game Server transmite `LiveMatchUpdate`, a plataforma envia `RefereeCommand`. Contrato forte, HTTP/2, um canal só nos dois sentidos. Alternativa descartada: polling REST (latência alta para "pausar agora") e Kafka (não é request/reply de baixa latência).

### D3. Alocação: gRPC unário

Contrato `.proto` versionado e baixa latência.

### D4. Mensageria assíncrona: Apache Kafka

| Critério | RabbitMQ | **Apache Kafka (escolhido)** |
|---|---|---|
| Volume no pico (~25.000 eventos/s de combate) | Suporta, mas perto do limite confortável de um cluster pequeno | Folgado |
| Ordem dos eventos de uma partida | Só com fila única por partida (inviável com 1.250 partidas) | Garantida por partição com chave `matchId` |
| Vários consumidores independentes do mesmo fluxo (IA, analytics, ranking) | Uma fila por consumidor, mensagens copiadas | Grupos de consumidores lendo o mesmo log |
| Reprocessar eventos antigos (novo modelo de IA, correção de bug) | Não (mensagem some após o consumo) | Sim, por retenção e reset de offset |
| Processamento em janelas deslizantes | Precisa ser construído | Natural sobre o log ordenado |
| Retry com atraso e DLQ | Nativo | Padrão de tópicos de retry (precisa ser montado) |
| Operação | Mais simples | Mais complexa (mitigado com KRaft, sem ZooKeeper) |

**Escolhido: Apache Kafka**, com **Schema Registry** (Avro) para versionar os contratos. Os fatores decisivos:
1. **Ordem por partida**: o agente de IA precisa ver os eventos de cada partida na ordem em que aconteceram para analisar janelas deslizantes.
2. **Replay**: quando o modelo de IA muda, reprocessamos os últimos 7 dias de `combat.windows` para medir a melhoria (avaliação contínua, ADR-006).
3. **Vários consumidores do mesmo fluxo** sem duplicar mensagens.

O ponto fraco (retry com atraso) é resolvido com tópicos `*.retry.1m`, `*.retry.10m` e `*.dlq`.

### D5. Cliente e plataforma: REST/JSON via API Gateway

OpenAPI, cacheável, conhecido. Pagamentos usam `Idempotency-Key`.

## Consequências

**Positivas**
- O Game Server publica e esquece: lentidão da plataforma ou da IA nunca afeta o tick (C1).
- O agente de IA recebe eventos ordenados com atraso de segundos, o que viabiliza alertas em tempo real (C5).
- Pagamentos confirmados nunca se perdem (log replicado, outbox).

**Negativas**
- Kafka é a peça mais complexa de operar da arquitetura.
- Entrega "pelo menos uma vez" exige consumidores idempotentes.
- WebSocket (TCP) sofre com perda de pacotes; mitigado pela interpolação de 100 ms.
