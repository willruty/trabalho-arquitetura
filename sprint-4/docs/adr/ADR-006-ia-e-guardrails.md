# ADR-006: Integração de Componentes Probabilísticos e Guardrails de IA

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-11-15 |
| Decisores | Equipe completa |
| Cenários relacionados | C1, C5 (QAW) |

## Contexto

Com torneios pagos, trapaça e conluio afetam dinheiro de jogadores honestos. O kata *Going... Going... Gone!* exige um agente de IA de fraude e conluio **em tempo real**. Mas componentes de IA são probabilísticos: erram, podem inventar evidências, custam por chamada e dependem de serviços externos. Um falso positivo significa punir um inocente; um falso negativo significa pagar prêmio a um fraudador.

## Alternativas

| Alternativa | Avaliação |
|---|---|
| A. Análise só pós-partida (em lote) | Simples, mas o alerta chega depois que a partida de torneio acabou; árbitro não consegue agir. |
| B. Só regras e estatística em tempo real | Rápido e barato, mas não captura padrões sutis e gera falsos positivos em jogadores habilidosos e em amigos. |
| C. LLM em tempo real para todas as janelas | Inviável em custo e latência (10.000 janelas por segundo no pico). |
| D. Classificador próprio treinado | Ótimo a longo prazo, mas ainda não temos milhares de casos rotulados. |
| **E. Processamento de fluxo + Model Tiering + RAG + humano no circuito (escolhida)** | Detecção em janelas deslizantes sobre o Kafka; tiers de custo crescente; casos revisados como contexto; humanos decidem. |

## Decisão

1. **Processamento de fluxo:** o Integrity Stream Processor consome `combat.windows` (chave `matchId`, ADR-003) e mantém janelas deslizantes de 60 s por jogador e por par de jogadores.
2. **Dois detectores:** individual (trapaça, 12 features) e relacional (conluio, grafo de interações: não agressão, foco compartilhado, reciprocidade de abates, coocorrência histórica, dispositivo em comum).
3. **Tiering:** tier 0 no Game Server, score no stream, tier 1 em SLM local (Ollama), tier 2 em Claude Opus 5.5 com effort baixo. Cada tier só recebe o que o anterior não resolveu.
4. **RAG** sobre casos revisados no pgvector, separados por tipo (trapaça ou conluio).
5. **Porta única** `ICheatAnalysisModel` (Application) com `OllamaAnalysisModel` e `ClaudeAnalysisModel`; trocar de modelo é trocar um adaptador (DIP). O encadeamento dos tiers usa **Chain of Responsibility** (`TieredCheatAnalyzer`).
6. **Guardrails obrigatórios** (documento 08): PII higienizada, nenhum texto do jogador no prompt, JSON Schema revalidado, coerência das evidências, evidência acumulada para conluio, orçamento de tokens, Circuit Breaker com fallback, auditoria, avaliação contínua com replay do Kafka.
7. **Humano no circuito:** a IA só alerta, abre casos e retém prêmios. Pausar, desclassificar e banir são exclusivos de humanos.

## Consequências

**Positivas**
- Primeiro alerta em ~5 s, permitindo ao árbitro agir durante a partida (C5).
- Nenhuma chamada de IA acontece dentro do Game Server; o tick nunca espera um modelo (C1).
- Custo proporcional à dificuldade dos casos, não ao número de jogadores.
- Cada revisão humana melhora o sistema (RAG); os casos rotulados viram, com o tempo, o conjunto de treino da alternativa D.

**Negativas e riscos**
- O Stream Processor tem estado em memória; rebalanceamentos exigem reconstrução a partir do log (2 min de replay).
- O SLM local exige GPU (custo fixo).
- O tier 2 envia dados anonimizados a um provedor externo: revisão de LGPD e termos de uso.
- Fraudadores se adaptam ao que é medido; as features precisam evoluir.
