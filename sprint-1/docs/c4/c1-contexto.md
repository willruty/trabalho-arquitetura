# C4 Nível 1: Diagrama de Contexto

Mostra o Rift Arena como uma caixa única, quem o usa e com quais sistemas externos ele conversa.

```mermaid
C4Context
    title Rift Arena: Diagrama de Contexto do Sistema

    Person(player, "Jogador", "Joga partidas e se inscreve em torneios")
    Person(ref, "Árbitro", "Acompanha partidas de torneio e recebe alertas da IA")
    Person(mod, "Moderador", "Revisa casos de trapaça e conluio")

    System(rift, "Rift Arena", "FPS competitivo no navegador: partidas em tempo real, torneios pagos, ranking e integridade com IA")

    System_Ext(pix, "PSP PIX", "Cobranças PIX, webhooks de confirmação e pagamento de prêmios")
    System_Ext(card, "Adquirente de Cartão", "Autorização e captura de cartão tokenizado")
    System_Ext(idp, "Provedor de Identidade", "Login social via OIDC")
    System_Ext(fed, "Federação de eSports", "Sistema legado SOAP/XML que recebe resultados oficiais")
    System_Ext(llm, "Provedor de LLM", "IA generativa em nuvem para casos ambíguos")

    Rel(player, rift, "Joga, paga inscrições", "HTTPS / WebSocket")
    Rel(ref, rift, "Acompanha e pausa partidas", "HTTPS / WebSocket")
    Rel(mod, rift, "Revisa casos", "HTTPS")
    Rel(rift, pix, "Cria cobranças, paga prêmios", "HTTPS / webhook")
    Rel(rift, card, "Autoriza pagamentos", "HTTPS")
    Rel(rift, idp, "Autentica usuários", "OIDC")
    Rel(rift, fed, "Reporta resultados de torneio", "SOAP/XML")
    Rel(rift, llm, "Envia telemetria anonimizada", "HTTPS")
```

## Elementos

| Elemento | Tipo | Descrição |
|---|---|---|
| Jogador | Pessoa | Usuário final, desktop com teclado e mouse (Pointer Lock API). |
| Árbitro | Pessoa | Equipe de torneios. Recebe alertas ao vivo e pode pausar uma partida. |
| Moderador | Pessoa | Equipe de operação. Decide sobre casos de trapaça e conluio. |
| Rift Arena | Sistema | O sistema em construção. |
| PSP PIX | Externo | Provedor de serviços de pagamento com API PIX. Subdomínio genérico. |
| Adquirente de Cartão | Externo | Processa cartões; o número do cartão é tokenizado no navegador e nunca toca nossos servidores. |
| Provedor de Identidade | Externo | Keycloak federado com Google e Discord. |
| Federação de eSports | Externo (legado) | SOAP/XML, janela de manutenção semanal, instável. |
| Provedor de LLM | Externo | Usado apenas no tier 2 da IA, com dados anonimizados. |
