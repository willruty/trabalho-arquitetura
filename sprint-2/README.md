# Sprint 2: Domínio e Design de Código

**Período:** 16/09/2026 a 07/10/2026 | **Marco:** ADR-002 aprovado em 05/10/2026

## Objetivo da sprint

Modelar o Core Domain (Resolução de Combate) com DDD tático e definir como o código do servidor será desacoplado com SOLID e padrões GoF, de modo que novas armas e modos de jogo entrem sem alterar o núcleo (cenário C8 do QAW).

## Equipe e funções nesta sprint

| Integrante | Função na Sprint 2 | Responsabilidade principal |
|---|---|---|
| [Nome do Aluno 2] | Liderança | Modelo de domínio e linguagem ubíqua |
| [Nome do Aluno 1] | Pesquisador | Padrões GoF aplicáveis a jogos (State, Strategy, Observer) |
| [Nome do Aluno 5] | Pesquisador | Organização em camadas e DIP no .NET |
| [Nome do Aluno 3] | Executor | C4 nível 3 do Game Server |
| [Nome do Aluno 4] | Executor | Exemplos de código ilustrativos e ADR-002 |

## Entregáveis

| # | Entregável | Documento |
|---|---|---|
| 1 | Modelo de domínio, agregados e linguagem ubíqua | [docs/03-modelo-de-dominio.md](docs/03-modelo-de-dominio.md) |
| 2 | Princípios SOLID e catálogo de padrões GoF | [docs/04-solid-e-padroes-gof.md](docs/04-solid-e-padroes-gof.md) |
| 3 | C4 Nível 3: Componentes do Game Server | [docs/c4/c3-componentes-game-server.md](docs/c4/c3-componentes-game-server.md) |
| 4 | ADR-002: Desacoplamento com DIP e padrões GoF | [docs/adr/ADR-002-dip-e-padroes-gof.md](docs/adr/ADR-002-dip-e-padroes-gof.md) |

## Critérios de aceite da sprint

- [x] Agregado `Match` com invariantes explícitas.
- [x] Regra de ordenação estrita de disparos (o "motor de arremate" do jogo) definida e ilustrada.
- [x] Observer e Strategy desacoplados do motor de resolução (entregável do kata *Going... Gone!*).
- [x] Glossário da linguagem ubíqua com pelo menos 15 termos.
- [x] Pelo menos 4 padrões GoF justificados por um problema concreto (não "padrão pelo padrão").
- [x] Regra de dependência entre camadas definida (base para as fitness functions da Sprint 4).
- [x] C4 nível 3 do contêiner mais crítico (Game Server).

## Retrospectiva

**O que funcionou:** modelar o tempo como um parâmetro (`nowMs`) passado para o domínio, em vez de ler o relógio dentro das classes, deixou toda a simulação determinística e testável.

**O que melhorar:** a discussão sobre onde fica a física (domínio ou infraestrutura) consumiu tempo. Decisão registrada no ADR-002: a física é regra de negócio do jogo, então fica no domínio.

## Backlog levado para a Sprint 3

- Definir canal bidirecional e mensageria assíncrona com Kafka (ADR-003).
- Estratégia de Bulkhead para o gateway de pagamentos PIX/Cartão.
- Netcode: predição no cliente, interpolação e compensação de latência.
- Mapa de contexto completo (incluindo Torneios e Pagamentos) e Camada Anti-Corrupção (ADR-004).
- Padrões de resiliência (Circuit Breaker, Retry, Bulkhead, DLQ).
