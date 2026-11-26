# C4 Nível 2: Diagrama de Contêineres (versão 2)

Evolução da v1 com as decisões do ADR-003 (canal bidirecional e Kafka) e ADR-004 (contextos e ACLs). O agente de IA entra na versão final (Sprint 4).

```mermaid
C4Container
    title Rift Arena: Contêineres (v2, Sprint 3)

    Person(player, "Jogador")
    Person(ref, "Árbitro")

    System_Boundary(rift, "Rift Arena") {
        Container(web, "Web Client", "TypeScript, Three.js", "Jogo, inscrições, painel do árbitro")
        Container(gw, "API Gateway", "ASP.NET Core + YARP", "REST e WebSocket, JWT, rate limit, afinidade")
        Container(platform, "Platform API", "ASP.NET Core, monólito modular", "Identity, Matchmaking, Tournaments, Ranking, Integrity, Federation")
        Container(game, "Game Server", "ASP.NET Core, WebSockets, gRPC", "Motor de resolução de combate, N instâncias")
        Container(pay, "Payment Service", "ASP.NET Core", "PIX e cartão em compartimentos isolados (Bulkhead)")
        ContainerQueue(kafka, "Event Streaming", "Apache Kafka", "combat.windows, match.lifecycle, payments.events")
        ContainerDb(db, "Banco da Plataforma", "PostgreSQL", "Torneios, ratings, outbox")
        ContainerDb(paydb, "Banco de Pagamentos", "PostgreSQL", "Transações, idempotência, outbox")
        ContainerDb(redis, "Cache e Filas", "Redis", "Tickets, registro de Game Servers")
    }

    System_Ext(pix, "PSP PIX")
    System_Ext(card, "Adquirente de Cartão")
    System_Ext(fed, "Federação de eSports", "SOAP/XML")

    Rel(player, web, "Usa", "HTTPS")
    Rel(ref, web, "Arbitra", "HTTPS")
    Rel(web, gw, "REST e WebSocket bidirecional", "HTTPS / WSS")
    Rel(gw, platform, "/api/*", "HTTP")
    Rel(gw, game, "/ws/game/* com afinidade", "WebSocket")
    Rel(gw, pay, "/api/payments/*", "HTTP")
    Rel(platform, game, "AllocateMatch e MatchControl (stream bidirecional)", "gRPC")
    Rel(game, kafka, "Eventos de combate e de partida", "Kafka")
    Rel(pay, kafka, "Eventos de pagamento", "Kafka")
    Rel(kafka, platform, "Grupos de consumidores", "Kafka")
    Rel(platform, db, "SQL")
    Rel(pay, paydb, "SQL")
    Rel(platform, redis, "RESP")
    Rel(pay, pix, "Cobranças e prêmios (compartimento PIX)", "HTTPS")
    Rel(pix, pay, "Webhook de confirmação", "HTTPS")
    Rel(pay, card, "Autorizações (compartimento Cartão)", "HTTPS")
    Rel(platform, fed, "Relatórios via ACL", "SOAP/XML")
```

## Mudanças em relação à v1

| Mudança | Motivo |
|---|---|
| API Gateway (YARP) | Ponto único de entrada; WebSocket com afinidade. Tecnologia formalizada no ADR-005. |
| gRPC unário e **bidirectional streaming** | Alocação de partidas e canal do árbitro (ADR-003). |
| **Apache Kafka** | Fluxo de eventos de combate em tempo real para a IA, ordem por partida, reprocessamento (ADR-003). |
| Compartimentos PIX e Cartão no Payment Service | Estratégia de Bulkhead (documento de resiliência). |
| Redis | Fila de matchmaking e registro de capacidade dos Game Servers. |
| ACLs em Federation e Payments | ADR-004. |
