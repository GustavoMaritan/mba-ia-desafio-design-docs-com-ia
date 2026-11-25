# ADR-005: Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Diego, Larissa, Sofia, Marcos

## Contexto

Com outbox, retry e um worker que pode cair entre "enviou" e "marcou como entregue", é inevitável que, em alguns casos, o mesmo evento seja enviado mais de uma vez. Precisamos assumir uma semântica de entrega e dar ao cliente uma forma de lidar com duplicatas.

## Decisão

- A garantia é **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado.
- Cada evento recebe um **UUID gerado no momento em que entra na outbox**, enviado no header **`X-Event-Id`** e no campo `event_id` do payload. O mesmo evento mantém o mesmo id em todas as tentativas e também no replay da DLQ.
- O cliente **deduplica pelo `event_id`** do lado dele.
- O produto vai documentar isso com destaque no portal de desenvolvedor.

## Alternativas consideradas

### Exactly-once
**Descartada:** exigiria coordenação dos dois lados (nós e o cliente) e ficaria muito mais complexo. At-least-once com id de evento é o padrão de mercado, usado por Stripe e GitHub.

## Consequências

**Positivas**
- O modelo é simples e compatível com outbox + retry.
- É um padrão conhecido pelos clientes, o que facilita a integração.

**Negativas / trade-offs**
- Parte da responsabilidade vai para o cliente, que precisa implementar a deduplicação ([09:25] Sofia).
- Um cliente que ignore o `X-Event-Id` pode processar o mesmo evento em dobro.

## Referências

- Transcrição: [09:24]–[09:26] (Diego, Bruno, Sofia, Marcos), [09:44] (Diego)
