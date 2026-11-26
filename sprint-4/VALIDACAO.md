# Validação da entrega do Rift Arena

Revisei a documentação da sprint 4 e confirmo a composição da nossa equipe. Eu, Caio Marani (@feudal911), sou o líder. Willian Rutyna (@willruty) e Kaua Peres (@pereszx) completam o grupo.

O projeto descreve a arquitetura de um FPS multiplayer que roda no navegador. Conferi como a proposta resolve três pontos da partida e dos torneios:

- **Combate.** O navegador envia as ações do jogador. O servidor ordena os disparos e decide os acertos e abates, inclusive quando dois jogadores atiram quase ao mesmo tempo.
- **Integridade.** Os eventos de combate passam pelo Kafka para que o agente de IA procure indícios de trapaça ou conluio durante as partidas.
- **Pagamentos.** As inscrições e premiações dos torneios ficam em um serviço separado, para que uma falha no PIX ou no cartão não interrompa uma partida.

Confirmo que essa é a arquitetura proposta pela equipe e que a documentação está de acordo com a nossa entrega. Da minha parte, tudo certo.

Caio Marani (@feudal911), líder da equipe
