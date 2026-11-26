# C4 Nível 2: Diagrama de Contêineres (versão 1)

Primeira versão, derivada do ADR-001. Mostra os três artefatos de deploy: Platform API (monólito modular), Game Server e Payment Service. Gateway, Kafka e IA entram nas Sprints 3 e 4 (versão final em `sprint-4/docs/c4/c2-containers-final.md`).

```mermaid
C4Container
    title Rift Arena: Contêineres (v1, Sprint 1)

    Person(player, "Jogador")

    System_Boundary(rift, "Rift Arena") {
        Container(web, "Web Client", "TypeScript, Three.js, Vite", "Renderiza a arena 3D, captura entradas, prediz o próprio movimento")
        Container(platform, "Platform API", "ASP.NET Core, monólito modular", "Identidade, matchmaking, torneios, ranking, integridade, federação")
        Container(game, "Game Server", "ASP.NET Core, WebSockets", "Motor de resolução de combate a 30 Hz")
        Container(pay, "Payment Service", "ASP.NET Core", "Inscrições e prêmios via PIX e cartão, isolado")
        ContainerDb(db, "Banco da Plataforma", "PostgreSQL", "Jogadores, torneios, ratings")
        ContainerDb(paydb, "Banco de Pagamentos", "PostgreSQL", "Transações, idempotência")
    }

    System_Ext(pix, "PSP PIX")
    System_Ext(card, "Adquirente de Cartão")
    System_Ext(fed, "Federação de eSports", "SOAP/XML")

    Rel(player, web, "Usa", "HTTPS")
    Rel(web, platform, "Login, fila, torneios", "REST/JSON")
    Rel(web, game, "Entradas e snapshots", "WebSocket")
    Rel(web, pay, "Paga inscrição", "REST/JSON")
    Rel(platform, game, "Aloca partida", "a definir (ADR-003)")
    Rel(game, platform, "Eventos da partida", "a definir (ADR-003)")
    Rel(pay, platform, "Pagamento confirmado", "a definir (ADR-003)")
    Rel(platform, db, "Lê e grava", "SQL")
    Rel(pay, paydb, "Lê e grava", "SQL")
    Rel(pay, pix, "Cobranças e prêmios", "HTTPS")
    Rel(pay, card, "Autorizações", "HTTPS")
    Rel(platform, fed, "Resultados", "SOAP/XML")
```

## Contêineres

| Contêiner | Tecnologia | Responsabilidade | Escala |
|---|---|---|---|
| Web Client | TypeScript, Three.js, Vite | Renderização, entrada, predição, interpolação | CDN (estático) |
| Platform API | ASP.NET Core | Regras de negócio fora da partida | Horizontal, sem estado |
| Game Server | ASP.NET Core + WebSockets | Motor de resolução, estado em memória | Horizontal por partidas, com estado |
| Payment Service | ASP.NET Core | Orquestração de pagamentos, isolada | Horizontal, sem estado |
| Bancos | PostgreSQL | Persistência (bancos separados para pagamentos) | Primário + réplica |

## Pontos em aberto (levados para as próximas sprints)

- Protocolos entre os contêineres e broker de mensagens: ADR-003.
- Ponto único de entrada para o cliente (gateway): ADR-005.
- Onde roda o agente de IA e como ele recebe eventos em tempo real: ADR-006.
