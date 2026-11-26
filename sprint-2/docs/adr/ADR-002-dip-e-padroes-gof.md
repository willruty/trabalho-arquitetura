# ADR-002: Estratégia de Desacoplamento de Código com DIP e Padrões GoF

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-10-05 |
| Decisores | Equipe completa |
| Cenários relacionados | C1, C2, C8 (QAW) |

## Contexto

O núcleo do Rift Arena (Resolução de Combate) precisa:

1. Receber novas armas, modos de jogo e reações a eventos com frequência, sem regressões no que já funciona (C8).
2. Resolver disparos em ordem estrita e de forma reproduzível (C2).
3. Ser testável de forma determinística: um bug de "acerto que não contou" precisa ser reproduzido num teste unitário.
4. Rodar dentro de um orçamento de 33 ms por tick, sem I/O no caminho (C1).

Um protótipo inicial, discutido em sala, concentrava tudo numa classe `GameManager` com `switch` por tipo de arma e por fase da partida. Cada nova arma exigia alterar esse `switch` em três lugares.

## Alternativas consideradas

| Alternativa | Descrição | Avaliação |
|---|---|---|
| A. Classe central com condicionais | `GameManager` com `switch` por arma e fase | Simples no início, viola OCP, difícil de testar, conflitos de merge constantes. |
| B. Entity Component System (ECS) | Dados em componentes, lógica em sistemas (padrão comum em engines) | Excelente desempenho com milhares de entidades, mas temos no máximo 8 por partida; curva de aprendizado alta e afasta o modelo da linguagem ubíqua. |
| C. Camadas com DIP + padrões GoF no domínio (escolhida) | Domínio puro com Strategy, State, Factory Method e Observer; portas para I/O | Modelo próximo da linguagem do negócio, extensível, testável. |

## Decisão

1. **Clean Architecture por contexto:** Domain, Application, Infrastructure e Host, com dependências apontando para dentro. O Domain não referencia nenhum pacote externo.
2. **Portas e adaptadores (DIP):** todo I/O (broker, WebSocket, relógio, aleatoriedade) é uma interface declarada no Domain ou Application.
3. **Tempo e aleatoriedade injetados:** `nowMs` como parâmetro e `IRandomSource` com semente, para simulação determinística.
4. **Padrões GoF no domínio:**
   - **Strategy** para armas (`IWeaponStrategy`).
   - **State** para o ciclo de vida da partida (`MatchState`).
   - **Factory Method** para criar partidas por modo (`MatchCreator`).
   - **Observer** para eventos de domínio (`IMatchObserver`), com a regra de que observadores não fazem I/O.
5. **A física é domínio.** Movimento, colisão e raycast são regras do jogo (definem o que é justo), então ficam em `Rift.Game.Domain`.

## Consequências

**Positivas**
- Nova arma = nova classe `IWeaponStrategy` + registro no modo. Nova regra de fase = novo `MatchState`.
- Testes do domínio rodam em milissegundos, sem rede nem broker.
- A regra de dependência pode ser verificada automaticamente (fitness functions, Sprint 4).

**Negativas**
- Mais arquivos e interfaces do que a solução com condicionais.
- Chamadas por interface têm custo marginal; irrelevante na escala de 8 combatentes, mas `Vec3` e `HitInfo` são `readonly struct` para evitar alocações no laço.
- O Observer síncrono exige disciplina (nada de I/O dentro dele). Mitigação: revisão de código e um teste de arquitetura que proíbe observadores do Domain/Application de dependerem de tipos de rede.
