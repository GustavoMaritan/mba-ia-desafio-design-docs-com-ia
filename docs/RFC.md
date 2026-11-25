# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Gustavo Maritan, a partir da reunião técnica conduzida por Larissa |
| **Status** | Em revisão |
| **Data** | Semana da reunião técnica (quinta-feira, 09:00) |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) |

## 1. TL;DR

Propomos notificar os clientes B2B por **webhook outbound** sempre que o status de um pedido mudar. O evento é gravado numa **outbox no MySQL dentro da mesma transação** da mudança de status. Um **worker em processo separado** lê a outbox a cada 2 segundos e faz o POST assinado com **HMAC-SHA256** (secret por endpoint). Se o envio falhar, o worker tenta de novo com **backoff de 1m/5m/30m/2h/12h** e, esgotadas as tentativas, move o evento para uma **DLQ** com replay manual por admin. A entrega é **at-least-once**, e o cliente deduplica pelo `X-Event-Id`. Tudo isso reaproveita os padrões que já existem no projeto e não exige infraestrutura nova.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo hoje descobrem mudanças de status fazendo polling em `GET /orders`, o que deixa a integração lenta e cara para eles. Os três pediram notificação "em tempo real", definida pelo produto como **menos de 10 segundos**. A Atlas sinalizou que pode migrar para um concorrente se isso não sair até o fim do trimestre.

O sistema atual não tem nenhum mecanismo de evento, fila ou notificação externa. A mudança de status vive em `OrderService.changeStatus`, numa transação que já atualiza `orders`, grava `order_status_history` e mexe em estoque. Qualquer solução precisa:

- não deixar essa transação mais lenta nem dependente de sistemas externos;
- não perder evento quando o status muda, nem gerar evento quando a mudança é desfeita;
- ser operável por um time pequeno.

## 3. Proposta

### 3.1 Visão geral

```
PATCH /orders/:id/status
        │
        ▼
OrderService.changeStatus ── $transaction ─────────────────────────┐
   update orders + insert order_status_history + estoque           │
   publishWebhookEvent(tx, ...) → INSERT webhook_outbox (snapshot) │
        └───────────────────────────────────────────── commit ─────┘
                                   │
                    worker (processo separado, poll 2s)
                                   │
              POST https://cliente/...  (HMAC, X-Event-Id, timeout 10s)
                  │ 2xx → entregue      │ falha → retry 1m/5m/30m/2h/12h
                                         └ esgotou → webhook_dead_letter
                                                      └ replay manual (ADMIN)
```

### 3.2 Componentes

- **Configuração de webhooks:** o cliente (via usuário autenticado da nossa API) cadastra URL `https`, escolhe quais status quer receber e recebe uma secret gerada por nós. Há CRUD completo, rotação de secret e consulta do histórico de entregas.
- **Outbox (`webhook_outbox`):** linha inserida na transação da mudança de status, com o payload já renderizado. Só é inserida se algum webhook ativo do customer quiser aquele status. Ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md) e [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md).
- **Worker:** entry-point `src/worker.ts` (`npm run worker`), mesmo banco e `PrismaClient` próprio, polling de 2s, single-worker. Ver [ADR-002](adrs/ADR-002-worker-separado-em-polling.md).
- **Retry e DLQ:** 5 retentativas com progressão fixa e depois tabela `webhook_dead_letter` com replay via endpoint admin. Ver [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md).
- **Segurança:** HMAC-SHA256 no corpo, secret por endpoint, rotação com 24h de convivência, TLS obrigatório e payload limitado a 64 KB. Ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).
- **Semântica de entrega:** at-least-once, com `X-Event-Id` estável por evento. Ver [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md).
- **Padrões do projeto:** módulo `src/modules/webhooks`, `AppError` com códigos `WEBHOOK_*`, Pino, middleware de erro e Zod. Ver [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md).

Contratos, tabelas, fluxos detalhados e matriz de erros estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| Alternativa | Por que foi descartada | Origem |
|---|---|---|
| **Disparo síncrono** dentro de `changeStatus` | Um cliente lento travaria a mudança de status de outros pedidos, e não há o que fazer com a transação se o cliente estiver fora do ar (rollback de status não faz sentido). | [09:04] Bruno, [09:06] Diego |
| **Redis Streams / fila externa** | Exige subir e operar infra nova (Redis Cluster). Para um time pequeno é overengineering, e o MySQL existente resolve. | [09:07] Larissa, Diego |
| **Trigger no banco para acordar o worker** | O MySQL não tem `NOTIFY/LISTEN`, e a trigger não avisa processo externo. Polling de 2s já atende o limite de 10s. | [09:09] Bruno, Diego |
| **Retry indefinido** ou **só 3 tentativas** | Indefinido deixa evento pendurado para sempre. 3 tentativas cobrem só cerca de 30 minutos e não sobrevivem a uma manutenção de 2h que já aconteceu. | [09:15]–[09:16] Diego, Bruno |
| **Exactly-once** | Exigiria coordenação dos dois lados e muito mais complexidade. At-least-once com id é o padrão de mercado (Stripe, GitHub). | [09:25] Diego |
| **Secret global** | Um vazamento comprometeria todos os clientes. | [09:21] Sofia |

## 5. Questões em aberto

1. **Rate limiting de saída por cliente.** Se 50 pedidos de um cliente mudarem de status em um minuto, enviamos 50 chamadas. Ficou como "observar e decidir depois", e ainda não sabemos o gatilho para agir. ([09:38]–[09:39] Diego, Larissa)
2. **Escala para múltiplos workers.** Com mais de um worker, perde-se a ordenação por pedido. As opções levantadas (particionar por `order_id` ou lock pessimista) não foram decididas. ([09:12]–[09:13] Diego, Bruno)
3. **Arquivamento da outbox.** Linhas entregues seriam arquivadas "depois de 30 dias ou assim", mas isso está fora do escopo desta feature e não tem dono nem prazo. ([09:08] Diego)
4. **Endurecimento de permissões do CRUD.** Hoje qualquer role autenticada pode configurar webhooks. A Sofia indicou que isso pode ser endurecido "mais pra frente". ([09:37] Sofia)
5. **Notificação ao cliente sobre falhas recorrentes (email).** Fica para a próxima fase, depois de medir o impacto. ([09:37] Larissa)

## 6. Impacto e riscos

**Impacto no sistema existente**
- `src/modules/orders/order.service.ts`: `changeStatus` passa a chamar `publishWebhookEvent(tx, ...)`. Uma falha nessa inserção faz rollback da mudança de status, de propósito.
- Novo processo em produção (worker), com deploy e monitoramento próprios.
- Quatro tabelas novas no MySQL: configuração, outbox, histórico de entregas e DLQ.

**Riscos principais** (detalhes de probabilidade e mitigação no [PRD](PRD.md#10-riscos-e-mitigação))
- Crescimento da outbox sem arquivamento definido.
- Duplicidade de eventos no cliente se ele ignorar o `X-Event-Id`.
- Vazamento de secret: mitigado pela secret por endpoint e pela rotação.
- Worker parado acumulando eventos: com single-worker não há redundância.
- Prazo: estimativa de **três sprints**, incluindo **dois dias úteis de revisão de segurança** pela Sofia antes do deploy. ([09:46]–[09:47] Larissa, Sofia)

## 7. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling de 2s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Payload snapshot e filtro na inserção](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)
