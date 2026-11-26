# Sprint 1: Fundação e Visão Arquitetural

**Período:** 26/08/2026 a 16/09/2026 | **Marco:** ADR-001 aprovado em 10/09/2026

## Objetivo da sprint

Adaptar o kata *Going... Going... Gone!* para o Rift Arena, definir o problema, o domínio de negócio e os atributos de qualidade que vão guiar todas as decisões seguintes, e escolher o estilo arquitetural macro do sistema.

## Equipe e funções nesta sprint

| Integrante | Função na Sprint 1 | Responsabilidade principal |
|---|---|---|
| [Nome do Aluno 1] | Liderança | Condução do QAW, redação do ADR-001 |
| [Nome do Aluno 2] | Pesquisador | Pesquisa de arquiteturas de jogos multiplayer (servidor autoritativo, netcode) |
| [Nome do Aluno 3] | Pesquisador | Classificação de subdomínios e requisitos |
| [Nome do Aluno 4] | Executor | Diagramas C4 nível 1 e 2 (v1) |
| [Nome do Aluno 5] | Executor | Cenários de qualidade e métricas da matriz QAW |

## Entregáveis

| # | Entregável | Documento |
|---|---|---|
| 1 | Visão do produto, desafio de engenharia, requisitos e subdomínios | [docs/01-visao-e-dominio.md](docs/01-visao-e-dominio.md) |
| 2 | Quality Attribute Workshop (QAW) e árvore de utilidade | [docs/02-qaw.md](docs/02-qaw.md) |
| 3 | C4 Nível 1: Diagrama de Contexto | [docs/c4/c1-contexto.md](docs/c4/c1-contexto.md) |
| 4 | C4 Nível 2: Diagrama de Contêineres (versão 1) | [docs/c4/c2-containers-v1.md](docs/c4/c2-containers-v1.md) |
| 5 | ADR-001: Estilo arquitetural | [docs/adr/ADR-001-estilo-arquitetural.md](docs/adr/ADR-001-estilo-arquitetural.md) |

## Critérios de aceite da sprint

- [x] Desafio principal de engenharia descrito em uma frase e justificado com números (concorrência, latência, escala).
- [x] Pelo menos 5 atributos de qualidade com cenário completo (fonte, estímulo, artefato, ambiente, resposta, medida).
- [x] Subdomínios classificados em Core, Supporting e Generic.
- [x] C4 nível 1 com todos os atores e sistemas externos.
- [x] ADR-001 com no mínimo duas alternativas comparadas e trade-offs explícitos.

## Retrospectiva

**O que funcionou:** mapear cada elemento do kata original para o jogo (lance para disparo, motor de arremate para motor de resolução, conluio de lances para conluio entre jogadores) deu à equipe um vocabulário comum. Começar pelos números (jogadores, ticks, banda) deixou claro por que o servidor de jogo e os pagamentos precisam ser separados do resto.

**O que melhorar:** o C4 nível 2 desta sprint ainda não mostra Kafka nem gateway. Isso é intencional (decisões de integração ficam para a Sprint 3), mas precisa ser revisitado.

## Backlog levado para a Sprint 2

- Modelar o domínio de combate (partida, jogador, armas), a regra de ordenação estrita de disparos e a linguagem ubíqua.
- Decidir como aplicar DIP e padrões GoF no código do servidor de jogo (ADR-002).
- Detalhar o C4 nível 3 do Game Server.
