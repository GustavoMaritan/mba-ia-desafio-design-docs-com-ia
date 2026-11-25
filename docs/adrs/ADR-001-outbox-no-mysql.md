# ADR-001: Padrão Outbox no MySQL para eventos de webhook

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status dos pedidos muda. Hoje eles fazem polling em `GET /orders`.

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já faz bastante coisa: atualiza `orders`, insere em `order_status_history` e mexe em `stock_quantity` dos produtos quando a transição pede.

Precisamos decidir **como o evento sai dessa transação** sem deixar a mudança de status lenta e sem criar inconsistência entre "o status mudou" e "o evento foi registrado".

## Decisão

Usar o **padrão Outbox na própria base MySQL**:

- Dentro da mesma transação de `changeStatus`, inserir uma linha na tabela `webhook_outbox` com o evento.
- Se a transação commitar, o evento está registrado. Se der rollback, o evento some junto. Se a inserção na outbox falhar, a transação inteira faz rollback.
- Um worker separado lê a outbox e faz as chamadas HTTP (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- A tabela tem índice em `status` (pendente, processando, falhou, entregue) e em `created_at`.

## Alternativas consideradas

### Disparo síncrono no service de orders
Fazer o HTTP para o cliente dentro de `changeStatus`.
**Descartada:** um cliente lento travaria a mudança de status de outros pedidos, e não existe resposta boa para "cliente fora do ar": dar rollback na mudança de status não faz sentido.

### Redis Streams (ou fila externa equivalente)
**Descartada:** exigiria subir e operar infraestrutura nova (Redis Cluster). Para um time pequeno, foi considerado overengineering, já que o MySQL existente resolve.

## Consequências

**Positivas**
- Atomicidade garantida entre mudança de status e registro do evento, sem 2PC nem infra nova.
- A transação de status não depende da disponibilidade de nenhum cliente externo.
- A outbox serve de registro auditável do que foi gerado.

**Negativas / trade-offs**
- A tabela cresce continuamente. O arquivamento de linhas entregues (algo como 30 dias) foi citado, mas ficou **fora do escopo** desta feature.
- O banco transacional passa a ser também o "broker", com mais escrita e mais leitura de polling na mesma instância. Aceitamos pelo volume atual.
- A latência passa a depender do ciclo do worker, e não é mais imediata.

## Referências

- Transcrição: [09:03]–[09:08] (Larissa, Bruno, Diego), [09:40]–[09:41] (Bruno, Diego)
- Código: `src/modules/orders/order.service.ts` (`changeStatus`), `prisma/schema.prisma`
