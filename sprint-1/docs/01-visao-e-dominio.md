# Visão do Produto e Domínio de Negócio

## 1. Contexto do projeto

**Architectural Kata:** *Going... Going... Gone!* (Leilões Online em Tempo Real), adaptado pela equipe para um FPS competitivo no navegador: o **Rift Arena**.

No kata original, milhares de participantes enviam lances em milissegundos, o sistema precisa ordená-los de forma estrita para decidir quem arremata, dinheiro real circula por um gateway de pagamentos, e um agente de IA procura fraudes e conluios entre participantes. O Rift Arena tem exatamente a mesma natureza:

| Característica do kata | Como aparece no Rift Arena |
|---|---|
| Eventos em milissegundos com ordenação estrita | Disparos de até 8 jogadores chegando a cada 33 ms; o servidor decide quem atirou primeiro |
| Motor de arremate | Motor de resolução de combate: decide acertos e abates de forma autoritativa |
| Participantes conectados em tempo real | Canal bidirecional (WebSocket) com cada jogador |
| Dinheiro em jogo | Torneios com inscrição paga e premiação via PIX e cartão |
| Fraude e conluio | Trapaça (aimbot, speedhack) e conluio (jogadores que se protegem, alimentação de abates) |

**Produto:** o jogador abre um link, entra na fila e em poucos segundos está numa arena 3D contra outros 7 jogadores. Ele pode jogar partidas casuais ou se inscrever em **torneios** (pagando a inscrição por PIX ou cartão) que distribuem prêmios aos primeiros colocados. Durante cada partida, um agente de IA observa o fluxo de eventos de combate em busca de trapaças e conluios. Ao fim, o ranking é atualizado, resultados oficiais vão para a federação de eSports parceira e prêmios só são pagos depois da liberação de integridade.

## 2. Desafio principal de engenharia

> Estruturar a arquitetura de **disparos em milissegundos com ordenação estrita** num servidor autoritativo, integrada a um **Agente de IA para Detecção de Trapaças e Conluios em Tempo Real**, escalando de 1.000 para 10.000 jogadores simultâneos e mantendo o **gateway de pagamentos PIX/Cartão isolado** para que nenhuma falha financeira afete as partidas (e vice-versa).

Esse desafio se desdobra em cinco problemas:

| Problema | Por que é difícil |
|---|---|
| Ordenação estrita | Disparos de jogadores diferentes chegam fora de ordem e com atrasos distintos. Se A e B atiram um no outro no mesmo tick, o servidor precisa decidir de forma determinística quem atirou primeiro, como o motor de arremate decide o lance vencedor. |
| Latência e justiça | Um jogador com 150 ms de ping vê o mundo "no passado". Sem compensação de latência, ele nunca acerta ninguém. |
| Escala elástica | O servidor de jogo guarda estado em memória e consome CPU de forma contínua. Não escala como uma API sem estado. |
| Integridade em tempo real | Num torneio com prêmio, descobrir um conluio depois que o prêmio foi pago é tarde demais. A detecção precisa acontecer durante a partida, sem punir inocentes de forma automática. |
| Isolamento financeiro | PSPs de PIX e adquirentes de cartão ficam lentos ou fora do ar. Isso não pode travar threads do jogo nem da plataforma (Bulkhead). |

## 3. Estimativas de dimensionamento (back-of-the-envelope)

| Grandeza | Valor | Raciocínio |
|---|---|---|
| Jogadores simultâneos (normal / pico) | 1.000 / 10.000 | Pico em finais de torneio |
| Jogadores por partida | 8 | Todos contra todos |
| Partidas simultâneas no pico | 1.250 | 10.000 / 8 |
| Taxa de simulação (tick) | 30 Hz | Orçamento de 33 ms por tick |
| Entradas recebidas no pico | 300.000 msg/s | 10.000 x 30 |
| Eventos de combate publicados no Kafka | ~25.000 eventos/s | Agregado por jogador a cada segundo + abates e danos |
| Banda de saída por cliente | ~12 KB/s | Snapshot de ~400 B x 30 Hz |
| Banda de saída total no pico | ~120 MB/s | 10.000 x 12 KB/s |
| Partidas por instância de Game Server | 50 (400 jogadores) | Estimativa para 2 vCPU, validada por teste de carga |
| Instâncias de Game Server no pico | ~25 | 1.250 / 50 |
| Inscrições pagas no pico | ~50 por segundo | Abertura de inscrições de um torneio grande |

## 4. Requisitos funcionais

| ID | Requisito |
|---|---|
| RF01 | O jogador entra como convidado (apelido) ou com conta (login social). Torneios exigem conta verificada. |
| RF02 | O jogador entra numa fila e é alocado numa partida com jogadores de habilidade próxima. |
| RF03 | O jogador se movimenta, mira e atira numa arena 3D. |
| RF04 | O servidor resolve disparos em ordem estrita: o disparo mais cedo vence, e jogador abatido não abate. |
| RF05 | Pelo menos 3 armas (fuzil, escopeta, railgun) e 2 modos (Deathmatch e Instagib). |
| RF06 | A partida tem ciclo de vida: aguardando, aquecimento, em andamento, encerrada. |
| RF07 | O jogador vê placar, killfeed, vida, munição e tempo. |
| RF08 | O jogador se inscreve em torneios pagando por PIX ou cartão; vagas são limitadas. |
| RF09 | Prêmios são pagos via PIX aos colocados após liberação de integridade. |
| RF10 | O agente de IA monitora partidas em tempo real e alerta trapaças e conluios. |
| RF11 | Em partidas de torneio, alertas chegam ao vivo para um árbitro, que pode pausar a partida. |
| RF12 | Moderadores revisam casos e decidem (confirmar ou descartar); só humanos punem. |
| RF13 | O ranking (Elo) é atualizado ao fim de cada partida ranqueada. |
| RF14 | Resultados de torneio são enviados à Federação de eSports (sistema legado). |

## 5. Requisitos não funcionais (resumo, detalhados no QAW)

| ID | Requisito |
|---|---|
| RNF01 | Processamento de entrada no servidor com p95 < 50 ms; ordenação determinística de disparos. |
| RNF02 | APIs REST com p95 < 250 ms. |
| RNF03 | Queda do PSP PIX ou da adquirente não indisponibiliza a interface nem afeta partidas. |
| RNF04 | Escalar de 1.000 para 10.000 jogadores com novas instâncias em menos de 45 s. |
| RNF05 | Alerta de trapaça ou conluio em menos de 10 s em partidas de torneio; nenhuma punição automática. |
| RNF06 | Adicionar arma, modo ou regra de reação a eventos sem alterar o motor de resolução. |

## 6. Classificação dos subdomínios (DDD estratégico)

| Tipo | Subdomínio | Justificativa |
|---|---|---|
| **Core** | **Resolução de Combate** (física, ordenação estrita de disparos, compensação de latência, regras de partida) | É o "motor de arremate" do produto. Uma resolução injusta ou lenta mata o jogo. |
| **Core** | **Integridade** (trapaça e conluio em tempo real) | Com prêmios em dinheiro, integridade é o que dá credibilidade aos torneios. |
| Supporting | Torneios | Inscrições, vagas, chaves e premiação. Específico do negócio, mas não é o diferencial. |
| Supporting | Matchmaking | Faixa de Elo que se expande com o tempo de espera. |
| Supporting | Ranking (Elo) | Regra conhecida. |
| Supporting | Federação (integração com legado) | Traduz resultados para o parceiro. Isolado por ACL. |
| Generic | Pagamentos (PIX/Cartão) | Resolvido com PSP e adquirente terceirizados; nossa parte é orquestrar e isolar. |
| Generic | Identidade | Keycloak (OIDC). |
| Generic | Observabilidade, Mensageria | OpenTelemetry, Apache Kafka. |

## 7. Atores e sistemas externos

| Ator / Sistema | Papel |
|---|---|
| Jogador | Joga partidas e se inscreve em torneios. |
| Árbitro | Acompanha partidas de torneio ao vivo e recebe alertas da IA. |
| Moderador | Revisa casos de integridade e decide. |
| PSP PIX | Gera cobranças PIX, confirma pagamentos (webhook) e executa os pagamentos de prêmios. |
| Adquirente de Cartão | Autoriza e captura pagamentos com cartão (cartão tokenizado no navegador). |
| Federação de eSports (legado) | Recebe resultados oficiais via SOAP/XML. Instável e lenta. |
| Provedor de Identidade | Login social (OIDC). |
| Provedor de LLM | Análise de casos ambíguos, com dados anonimizados. |

## 8. Resumo da solução proposta

A solução separa o sistema onde o perfil de carga muda. O **Game Server** mantém cada partida em memória e roda um laço de simulação a 30 Hz; ele é o **motor de resolução de combate**: único dono da verdade sobre posições, acertos e abates. Dentro de cada tick, os disparos são ordenados pelo instante em que foram feitos (corrigido pela latência de cada jogador) e resolvidos um a um; um jogador abatido por um disparo anterior no mesmo tick tem o próprio disparo anulado. O Game Server escala horizontalmente por partidas.

Identidade, matchmaking, torneios, ranking e federação vivem na **Platform API**, um monólito modular sem estado. **Pagamentos** ficam num serviço próprio, isolado por Bulkhead (processo, pools de conexão e limites de concorrência separados para PIX e cartão), tanto pelo isolamento de falhas quanto para reduzir o escopo de conformidade (PCI-DSS). Toda comunicação assíncrona passa pelo **Apache Kafka**: o Game Server publica eventos de combate e de fim de partida sem esperar ninguém.

O **agente de IA** consome o fluxo de eventos de combate em janelas deslizantes, durante a partida. Regras e estatística filtram, um modelo local barato analisa os suspeitos e só os casos ambíguos chegam a um LLM em nuvem. Em torneios, alertas chegam ao árbitro em segundos e prêmios ficam retidos até a liberação de integridade. A IA nunca pune: ela alerta e abre casos para humanos decidirem.
