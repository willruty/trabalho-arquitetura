# C4 Nível 3: Componentes do Game Server

O Game Server é o contêiner mais crítico (cenários C1 e C6). Este diagrama mostra seus componentes internos.

```mermaid
C4Component
    title Game Server: Componentes

    Container_Ext(web, "Web Client", "Three.js")
    Container_Ext(platform, "Platform API", "Matchmaking")
    ContainerQueue_Ext(mq, "Apache Kafka", "Eventos de combate e de integração")

    Container_Boundary(gs, "Game Server") {
        Component(ws, "Realtime Endpoint", "ASP.NET Core WebSockets", "Valida o token de entrada, recebe entradas, envia snapshots")
        Component(alloc, "Allocation Service", "gRPC", "Cria partidas a pedido do matchmaking")
        Component(loop, "Game Loop Service", "BackgroundService", "Dispara o tick de todas as salas a 30 Hz")
        Component(room, "Match Room", "Application", "Fila de comandos por partida, conexões, snapshot")
        Component(domain, "Combat Domain", "Domain", "Match, Fighter, armas, estados, física, compensação de latência")
        Component(obs, "Observers", "Application", "Buffer de eventos, telemetria, publicador de resultado")
        Component(pub, "Integration Publisher", "Infrastructure", "Outbox em memória + envio ao Kafka")
    }

    Rel(web, ws, "Entradas / snapshots", "WebSocket binário")
    Rel(platform, alloc, "AllocateMatch", "gRPC")
    Rel(ws, room, "Enfileira comandos")
    Rel(loop, room, "Tick(nowMs)")
    Rel(room, domain, "Aplica comandos e avança a simulação")
    Rel(domain, obs, "Notifica eventos", "Observer")
    Rel(obs, pub, "Enfileira MatchEnded")
    Rel(pub, mq, "Publica", "AMQP")
```

## Componentes

| Componente | Camada | Responsabilidade |
|---|---|---|
| Realtime Endpoint | Host | Aceita conexões WebSocket, valida o *join token* emitido pelo matchmaking, desserializa mensagens e as coloca na fila da sala. Nunca altera o domínio diretamente. |
| Allocation Service | Host | Recebe `AllocateMatch(mode, players)` via gRPC, usa o `MatchCreator` do modo e registra a sala. |
| Game Loop Service | Host | Laço de tempo fixo (30 Hz). Mede o próprio atraso e expõe a métrica `tick_duration_ms`. |
| Match Room | Application | Isola a concorrência: comandos chegam de várias threads numa `ConcurrentQueue`, mas só a thread do tick altera o `Match`. Monta e envia snapshots individuais (cada jogador recebe seu próprio ack). |
| Combat Domain | Domain | Todas as regras de jogo. Determinístico, sem I/O. |
| Observers | Application | Reagem aos eventos de domínio sem bloquear o tick. |
| Integration Publisher | Infrastructure | Esvazia a outbox em memória fora do laço e publica no Kafka com retry. |

## Fluxo de um tick

```mermaid
sequenceDiagram
    participant L as Game Loop
    participant R as Match Room
    participant M as Match (Domain)
    participant O as Observers
    participant C as Clientes
    L->>R: Tick(nowMs)
    R->>M: aplica entradas e disparos enfileirados
    M->>M: estado atual (State).Tick
    M->>M: move combatentes (até 3 entradas cada)
    M->>M: resolve disparos com rebobinamento
    M-->>O: ShotFired, FighterKilled...
    R->>R: SnapshotBuilder monta o estado
    R-->>C: snapshot individual (com ack de cada um)
```
