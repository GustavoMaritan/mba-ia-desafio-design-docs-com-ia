# ADR-007: Payload renderizado na inserção (snapshot) e filtro de eventos antes de gravar na outbox

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decidido no fim da call, depois do resumo)
- **Decisores:** Larissa, Diego, Bruno

## Contexto

Duas perguntas de modelagem da outbox ficaram para o final da reunião:

1. A outbox guarda o payload já pronto ou só o `order_id`, montando o payload na hora do envio?
2. O filtro de eventos (cada webhook escolhe quais status quer receber) é aplicado ao gravar na outbox ou ao enviar?

## Decisão

1. **Snapshot na inserção:** o payload JSON é montado no momento em que o evento entra na outbox, dentro da transação de `changeStatus`. Assim o evento reflete o estado do pedido no instante da mudança de status.
2. **Filtro na inserção:** se nenhum webhook ativo do customer quer aquele status, **nenhuma linha é inserida**.
3. **Payload enxuto:** `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Itens não são enviados**: quem quiser detalhes consulta `GET /orders/:id`.
4. **Limite de 64 KB:** um evento acima disso **gera erro e não é enviado**. Não truncamos.

## Alternativas consideradas

### Guardar só o `order_id` e renderizar no envio
**Descartada:** se o pedido mudar entre a inserção e o envio (inclusive durante horas de retry), o evento passaria a mostrar um estado que não corresponde à transição notificada.

### Filtrar no momento do envio
**Descartada:** geraria linhas na outbox que nunca serão enviadas. Filtrar antes economiza escrita e espaço.

### Truncar payloads grandes
**Descartada:** um payload desse tamanho indica que algo está errado. É melhor falhar explicitamente.

## Consequências

**Positivas**
- O evento é imutável e fiel ao momento da transição, inclusive em retries e replays.
- A outbox só contém trabalho real.
- O payload pequeno fica bem longe do limite de 64 KB.

**Negativas / trade-offs**
- A transação de `changeStatus` ganha trabalho extra: buscar os webhooks do customer, filtrar e montar o JSON.
- Se o cliente alterar o filtro de eventos, a mudança só vale para eventos futuros, e não para os que já estão na outbox.
- Informações que não estão no snapshot (como itens) exigem chamada adicional do cliente.

## Referências

- Transcrição: [09:23]–[09:24] (Sofia, Diego, Larissa), [09:33]–[09:34] (Marcos, Diego, Bruno), [09:43]–[09:44] (Marcos, Diego, Bruno), [09:51]–[09:52] (Bruno, Larissa, Diego)
- Código: `prisma/schema.prisma` (model `Order`: `orderNumber`, `totalCents`, `customerId`), `src/modules/orders/order.service.ts`
