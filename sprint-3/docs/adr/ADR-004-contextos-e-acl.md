# ADR-004: Mapeamento de Contextos Delimitados e Camada Anti-Corrupção

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-10-28 |
| Decisores | Equipe completa |
| Cenários relacionados | C3, C7, C8 (QAW) |

## Contexto

O sistema tem oito contextos (Combat, Integrity, Tournaments, Matchmaking, Ranking, Federation, Payments, Identity) em três contêineres. Precisamos definir fronteiras e relacionamentos, e proteger o modelo interno de três sistemas externos com modelos próprios:

- **Federação de eSports:** SOAP/XML legado com campos abreviados (`QT_ABATES`), datas em texto, `S/N`, códigos numéricos e mudanças de contrato sem aviso.
- **PSP PIX:** status e códigos próprios (`"CONCLUIDA"`, `"ATIVA"`), webhooks com formato do provedor.
- **Adquirente de cartão:** status (`"authorized"`, `"captured"`) e códigos de recusa numéricos.

## Alternativas

| Alternativa | Avaliação |
|---|---|
| Conformist (usar os modelos externos direto) | Rápido, mas espalha XML e códigos de provedor pelo sistema; trocar de PSP vira refatoração geral. |
| Banco compartilhado entre contextos | Acopla esquemas e quebra a autonomia dos módulos. |
| **Contextos com schemas próprios + ACLs (escolhida)** | Cada contexto tem modelo e schema próprios; cada sistema externo entra por uma camada de tradução com Adapter. |

## Decisão

1. **Oito contextos delimitados.** Na Platform API, cada um é um módulo com Domain, Application e Infrastructure próprios e schema próprio no PostgreSQL (`tournaments`, `ranking`, `integrity`, `federation`, `matchmaking`, `identity`). Payments tem banco próprio no Payment Service.
2. **Relacionamentos** conforme o mapa de contexto: Published Language (eventos Kafka com schemas Avro), Customer/Supplier (gRPC e REST), Open Host Service (ratings), Shared Kernel mínimo (`PlayerId`), Conformist com o Keycloak.
3. **ACL da Federação:** porta `IOfficialResultGateway`, `FederationResultAdapter` (Adapter GoF), `OfficialResultTranslator` e `FederationSoapClient` com Polly.
4. **ACL de Pagamentos:** porta `IPaymentProvider`, `PixProviderAdapter` e `CardAcquirerAdapter` (Adapter GoF), `PaymentProviderFactory` (Factory Method) escolhendo o adaptador pelo método, cada um no seu compartimento de Bulkhead.
5. Tipos externos são `internal` aos namespaces `*.Infrastructure.Acl`; uma fitness function proíbe referências a eles de fora.

## Consequências

**Positivas**
- Mudança no XML da Federação ou troca de PSP altera um único adaptador.
- Domínios testáveis com gateways falsos.
- Um novo método de pagamento (por exemplo, carteira digital) é um novo Adapter registrado na factory.

**Negativas**
- Mais classes e uma tradução a cada chamada (custo desprezível).
- Mapeamento `PlayerId` para registro de atleta exige cadastro próprio.
- Schemas separados impedem `JOIN` entre contextos; relatórios cruzados exigem consultas compostas ou réplica analítica.
