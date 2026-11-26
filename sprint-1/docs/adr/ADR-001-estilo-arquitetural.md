# ADR-001: Adoção do Estilo Arquitetural (Monólito Modular vs. Microsserviços)

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-09-10 |
| Decisores | Equipe completa |
| Cenários relacionados | C1, C3, C4, C8 (QAW) |

## Contexto

O Rift Arena tem três cargas de trabalho com perfis diferentes:

1. **Resolução de combate:** processo com estado em memória, uso contínuo de CPU, laço de 30 Hz, sensível a qualquer pausa (100 ms de pausa congelam 8 jogadores).
2. **Plataforma:** identidade, matchmaking, torneios, ranking, integridade e federação. Requisições curtas, sem estado.
3. **Pagamentos:** chamadas a PSP e adquirente que podem demorar segundos ou falhar; dados financeiros sujeitos a PCI-DSS e auditoria.

A equipe tem 5 pessoas e 4 sprints. O pico é de 10.000 jogadores simultâneos.

## Alternativas consideradas

### A. Monólito tradicional

- **Prós:** deploy único, sem rede entre componentes.
- **Contras:** laço de simulação dividindo CPU e GC com requisições REST; uma adquirente lenta prendendo threads do mesmo processo que roda partidas; todo o sistema dentro do escopo PCI.

### B. Microsserviços completos

Identity, Matchmaking, Tournaments, Ranking, Integrity, Federation, Payments e Game Server como 8 serviços.

- **Prós:** isolamento máximo.
- **Contras:** 8 pipelines e bancos para 5 pessoas; ranking, matchmaking e torneios têm carga baixa e não precisam escalar sozinhos.

### C. Híbrido: Monólito Modular + Game Server + Payment Service (escolhida)

- **Platform API:** um deploy, módulos isolados por contexto (Domain, Application e Infrastructure próprios, comunicação só por contratos públicos ou eventos).
- **Game Server:** deploy separado, escalado por partidas.
- **Payment Service:** deploy separado, com banco próprio.

## Decisão

Adotamos a **alternativa C**. As fronteiras de deploy ficam onde há um motivo concreto:

| Fronteira | Motivo |
|---|---|
| Game Server | Perfil de carga oposto (com estado, CPU contínua) e necessidade de escala independente. |
| Payment Service | **Bulkhead físico**: falhas e lentidão de PSP e adquirente ficam confinadas num processo próprio; e escopo de conformidade (PCI-DSS, auditoria) restrito a um serviço pequeno. |
| Resto | Monólito modular: baixa carga, consistência transacional simples, uma base de código. |

Dentro da plataforma, a modularização deixa a porta aberta para extrair módulos depois (o candidato mais provável é Integrity, por causa do processamento de fluxo e do custo de IA).

## Comparação

| Critério | Monólito | Microsserviços | Híbrido (escolhido) |
|---|---|---|---|
| Escala independente da simulação | Não | Sim | Sim |
| Isolamento do laço de 30 Hz | Não | Sim | Sim |
| Isolamento de falhas de pagamento | Não | Sim | Sim |
| Escopo PCI reduzido | Não | Sim | Sim |
| Complexidade operacional | Baixa | Alta | Média |
| Adequação a uma equipe de 5 pessoas | Alta | Baixa | Alta |

## Consequências

**Positivas**
- Game Server escala de 1 a ~25 instâncias sem tocar no resto (C4).
- O laço de simulação não compete com REST nem com chamadas de pagamento (C1).
- Uma adquirente fora do ar não consome nenhuma thread da plataforma nem do jogo (C3).

**Negativas e riscos**
- Disciplina para que um módulo não acesse o interior de outro. Mitigação: fitness functions (Sprint 4).
- Três artefatos se comunicam pela rede, o que exige definir protocolos (ADR-003).
- Se uma instância de Game Server cai, as partidas dela são perdidas. Aceito (partidas de até 10 minutos); em torneio, a partida é refeita por decisão do árbitro.
