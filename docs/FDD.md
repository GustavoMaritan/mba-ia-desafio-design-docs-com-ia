# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Pronto para implementação (pendente revisão de segurança) |
| **Proposta e decisões** | [RFC](RFC.md) · [ADRs](adrs/README.md) |
| **Produto** | [PRD](PRD.md) |

> Este documento descreve **como construir**. O porquê de cada decisão está nos ADRs. Quando algo aqui é um detalhe de implementação que não foi discutido na reunião, está marcado como **(detalhe de implementação)** para ficar claro o que é decisão do time e o que é proposta deste documento.

---

## 1. Contexto e motivação técnica

Hoje a única forma de um cliente saber que um pedido mudou de status é fazer polling em `GET /api/v1/orders`. A aplicação (Node.js + TypeScript, Express, Prisma/MySQL) não tem eventos, filas nem notificação externa.

A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), numa `prisma.$transaction` que:

1. lê o pedido com os itens;
2. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
3. debita ou repõe estoque quando aplicável (`shouldDebitStock` / `shouldReplenishStock`);
4. atualiza `orders.status`;
5. insere em `order_status_history`.

A feature precisa pendurar a emissão do evento **dentro** dessa transação, sem fazer I/O externo nela ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

## 2. Objetivos técnicos

| # | Objetivo | Medida |
|---|---|---|
| OT-1 | Entregar a notificação em menos de 10s após a mudança de status, em condições normais | p95 de `status_changed → 2xx do cliente` < 10s |
| OT-2 | Nenhuma mudança de status commitada sem evento correspondente (quando há webhook interessado) | 0 divergências entre `order_status_history` e `webhook_outbox` em teste de integração |
| OT-3 | A transação de `changeStatus` não faz I/O de rede | Garantido por design (o HTTP só ocorre no worker) |
| OT-4 | Cobrir indisponibilidades do cliente de até cerca de 15h sem perda | 5 retentativas (1m/5m/30m/2h/12h) antes da DLQ |
| OT-5 | Toda requisição enviada é verificável pelo cliente | 100% dos envios com `X-Signature` HMAC-SHA256 |

## 3. Escopo e exclusões

**Dentro**
- CRUD de configuração de webhook por customer, com filtro de status.
- Geração e rotação de secret (grace period de 24h).
- Outbox transacional, worker em processo separado, retry, DLQ e replay admin.
- Histórico de entregas por webhook (últimas 100).
- Assinatura HMAC-SHA256 e headers de entrega.

**Fora**
- Webhooks **inbound** (cliente enviando para nós).
- **Email** de alerta quando o webhook do cliente falha: próxima fase.
- **Dashboard visual**: projeto separado do time de frontend.
- **Rate limiting** de saída: observar e decidir depois.
- **Arquivamento** de linhas entregues da outbox (cerca de 30 dias).
- **Múltiplos workers** em paralelo e ordenação global.
- Envio dos **itens** do pedido no payload.

---

## 4. Modelo de dados

Quatro tabelas novas em `prisma/schema.prisma`, todas com id **UUID `Char(36)`** como o resto do schema ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

```prisma
enum WebhookOutboxStatus {
  PENDING      // aguardando envio ou próxima tentativa
  PROCESSING   // reservado pelo worker
  DELIVERED    // cliente respondeu 2xx
  FAILED       // esgotou tentativas e foi para a DLQ
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)   // válida até previousSecretExpiresAt
  previousSecretExpiresAt DateTime?
  events                  Json                          // ex.: ["SHIPPED","DELIVERED"]
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer                @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id                  String              @id @default(uuid()) @db.Char(36)
  eventId             String              @db.Char(36)   // X-Event-Id, estável entre tentativas e replay
  webhookId           String              @db.Char(36)
  orderId             String              @db.Char(36)
  eventType           String              @db.VarChar(64) // "order.status_changed"
  payload             Json                                 // snapshot renderizado na inserção
  status              WebhookOutboxStatus @default(PENDING)
  attempts            Int                 @default(0)
  nextAttemptAt       DateTime            @default(now())
  processingStartedAt DateTime?
  lastError           String?             @db.VarChar(500)
  requestId           String?             @db.VarChar(64) // correlação com o PATCH de origem
  createdAt           DateTime            @default(now())
  updatedAt           DateTime            @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id           String   @id @default(uuid()) @db.Char(36)
  webhookId    String   @db.Char(36)
  outboxId     String   @db.Char(36)
  eventId      String   @db.Char(36)
  attempt      Int
  success      Boolean
  statusCode   Int?
  responseBody String?  @db.Text      // truncado (detalhe de implementação: 4 KB)
  errorCode    String?  @db.VarChar(64)
  durationMs   Int
  createdAt    DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  outboxId     String    @db.Char(36)
  eventId      String    @db.Char(36)
  webhookId    String    @db.Char(36)
  payload      Json
  reason       String    @db.VarChar(500)
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  @@index([webhookId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas:
- **Uma linha de outbox por endpoint de destino.** Cada endpoint tem secret e `X-Webhook-Id` próprios, então o envio é independente. O `eventId` é gerado por linha no momento da inserção. (detalhe de implementação)
- O `DELETE` de webhook é **lógico** (`active = false`) para preservar histórico de entregas e DLQ. (detalhe de implementação)
- `WebhookDelivery` atende ao requisito do histórico de entregas (sucesso/falha, payload, resposta e tempo de resposta). O payload vem da outbox via `outboxId`.

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Dentro de `OrderService.changeStatus`, depois do `orderStatusHistory.create` e ainda na mesma `tx`:

```ts
await publishWebhookEvent(tx, order, from, to, { requestId });
```

`publishWebhookEvent` (`src/modules/webhooks/webhook.publisher.ts`):

1. `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`.
2. Filtra os endpoints cujo `events` contém `to`. **Se nenhum quiser o status, retorna sem inserir nada** ([ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)).
3. Para cada endpoint, gera `eventId = uuid()` e monta o **snapshot**:

```json
{
  "event_id": "4f1c2b7e-8a0d-4c55-9b9e-2d7c1f3e6a10",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-20T14:03:12.418Z",
  "order_id": "b6a3a8f2-5d1e-4b8a-9f3c-0e7d2a1c4b55",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "0c9e1d44-7f2a-4e61-8b3d-5a6f7e8d9c01",
  "total_cents": 125000
}
```

4. `tx.webhookOutbox.createMany(...)` com `status = PENDING` e `nextAttemptAt = now()`.
5. **Qualquer erro sobe e a transação inteira faz rollback**: o status não muda sem o evento.

### 5.2 Processamento pelo worker

Entry-point `src/worker.ts` (`npm run worker`), loop a cada **2s** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)):

```
loop:
  recuperar órfãos: PROCESSING com processingStartedAt < now() - 60s → PENDING   (detalhe de implementação)
  batch = SELECT * FROM webhook_outbox
          WHERE status = 'PENDING' AND next_attempt_at <= NOW()
          ORDER BY created_at ASC
          LIMIT 20                                                              (detalhe de implementação)
  para cada evento (sequencial, mantém ordem por created_at):
    UPDATE ... SET status='PROCESSING', processing_started_at=NOW()
    endpoint = webhook_endpoints[evento.webhookId]
    se endpoint inativo → DLQ (WEBHOOK_INACTIVE)
    body = JSON.stringify(evento.payload)
    se bytes(body) > 64 KB → DLQ (WEBHOOK_PAYLOAD_TOO_LARGE), sem tentar enviar
    POST endpoint.url  (timeout 10s, headers da seção 6.2)
    registra WebhookDelivery (status, corpo truncado, durationMs)
    2xx         → status = DELIVERED
    outro caso  → fluxo de retry (5.3)
  sleep 2s
```

### 5.3 Retry

Falha = timeout de 10s, erro de rede/TLS ou resposta não-2xx.

| Tentativa | Quando | Espera desde a falha anterior |
|---|---|---|
| 1 (envio inicial) | logo após a inserção (≤ 2s) | — |
| 2 | após falha da 1 | 1 min |
| 3 | após falha da 2 | 5 min |
| 4 | após falha da 3 | 30 min |
| 5 | após falha da 4 | 2 h |
| 6 | após falha da 5 | 12 h |
| — | falha da 6 | vai para a DLQ |

São 5 retentativas, cerca de 14h36 entre a primeira falha e a última tentativa ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).

```ts
const BACKOFF = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000];
// attempts = número de envios já feitos
if (attempts <= BACKOFF.length) {
  status = 'PENDING';
  nextAttemptAt = now + BACKOFF[attempts - 1];
} else {
  moveToDeadLetter('WEBHOOK_MAX_ATTEMPTS_EXCEEDED');
}
```

### 5.4 DLQ e replay

**Ida para a DLQ** (na mesma transação):
1. `webhook_outbox.status = FAILED`, `lastError` preenchido.
2. `INSERT webhook_dead_letter` com `payload`, `reason` e `failedAt`.
3. Log `webhook_moved_to_dlq` em nível `warn`.

**Replay** (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`, só ADMIN):
1. Carrega o registro. Se não existir, retorna `WEBHOOK_DEAD_LETTER_NOT_FOUND`. Se já tiver `replayedAt`, retorna `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
2. Em transação: insere uma nova linha na outbox com o **mesmo `eventId` e o mesmo `payload`**, `status = PENDING`, `attempts = 0`. Marca `replayedAt` e `replayedById = req.user.id`.
3. Log de auditoria `webhook_dlq_replayed` com `userId`, `deadLetterId`, `eventId` e `webhookId`.

---

## 6. Contratos públicos

Base: `/api/v1` (mesmo prefixo de `src/app.ts`). Todas as rotas usam `authenticate`. Erros seguem o formato do `error.middleware.ts`:

```json
{ "error": { "code": "WEBHOOK_INVALID_URL", "message": "Webhook URL must use https", "details": [ ... ] } }
```

### 6.1 API de configuração

#### `POST /api/v1/webhooks` — cadastrar webhook

O `customerId` vai no body, **não vem do JWT**. O JWT é do usuário operador que representa o cliente.

Request:
```json
{
  "customerId": "0c9e1d44-7f2a-4e61-8b3d-5a6f7e8d9c01",
  "url": "https://api.atlascomercial.com.br/hooks/pedidos",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` (a **secret só aparece aqui** e na rotação):
```json
{
  "id": "9d2f6c1a-3b4e-4f5a-8c7d-1e2f3a4b5c6d",
  "customerId": "0c9e1d44-7f2a-4e61-8b3d-5a6f7e8d9c01",
  "url": "https://api.atlascomercial.com.br/hooks/pedidos",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_3q8Zk...v1",
  "createdAt": "2026-11-20T13:00:00.000Z"
}
```

| Status | Quando |
|---|---|
| 201 | Criado |
| 400 | `WEBHOOK_INVALID_URL` (não-https ou malformada), `WEBHOOK_INVALID_EVENTS`, `VALIDATION_ERROR` |
| 401 | Sem token / token inválido |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

#### `GET /api/v1/webhooks?customerId={uuid}` — listar webhooks do customer

Response `200 OK` (nunca retorna a secret):
```json
{
  "data": [
    {
      "id": "9d2f6c1a-3b4e-4f5a-8c7d-1e2f3a4b5c6d",
      "customerId": "0c9e1d44-7f2a-4e61-8b3d-5a6f7e8d9c01",
      "url": "https://api.atlascomercial.com.br/hooks/pedidos",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-20T13:00:00.000Z",
      "updatedAt": "2026-11-20T13:00:00.000Z"
    }
  ]
}
```
Status: `200`, `400` (`customerId` ausente ou inválido), `401`.

#### `PATCH /api/v1/webhooks/:id` — editar URL, eventos ou estado

Request (todos os campos opcionais):
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`: mesmo formato do item da listagem.
Status: `200`, `400` (`WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_EVENTS`), `401`, `404` (`WEBHOOK_NOT_FOUND`).
A alteração de `events` só vale para eventos futuros. Eventos já na outbox não mudam ([ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md)).

#### `DELETE /api/v1/webhooks/:id` — remover

Response `204 No Content`. Status: `204`, `401`, `404` (`WEBHOOK_NOT_FOUND`).
Remoção lógica: eventos pendentes desse endpoint vão para a DLQ com `WEBHOOK_INACTIVE` quando o worker os pegar.

#### `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Request: corpo vazio.
Response `200 OK`:
```json
{
  "id": "9d2f6c1a-3b4e-4f5a-8c7d-1e2f3a4b5c6d",
  "secret": "whsec_N7pQ2...x9",
  "previousSecretExpiresAt": "2026-11-21T13:00:00.000Z"
}
```
Status: `200`, `401`, `404` (`WEBHOOK_NOT_FOUND`), `409` (`WEBHOOK_INACTIVE`).

#### `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Retorna as **últimas 100** tentativas, da mais recente para a mais antiga.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "e1d2c3b4-a5f6-4789-90ab-cdef01234567",
      "eventId": "4f1c2b7e-8a0d-4c55-9b9e-2d7c1f3e6a10",
      "attempt": 1,
      "success": false,
      "statusCode": 503,
      "durationMs": 812,
      "errorCode": "WEBHOOK_DELIVERY_FAILED",
      "payload": { "event_id": "4f1c2b7e-...", "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "responseBody": "Service Unavailable",
      "createdAt": "2026-11-20T14:03:13.230Z"
    }
  ]
}
```
Status: `200`, `401`, `404` (`WEBHOOK_NOT_FOUND`).

#### `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar evento da DLQ

Middlewares: `authenticate` + `requireRole('ADMIN')`.
Request: corpo vazio.
Response `202 Accepted`:
```json
{
  "deadLetterId": "7a6b5c4d-3e2f-4a1b-9c8d-7e6f5a4b3c2d",
  "eventId": "4f1c2b7e-8a0d-4c55-9b9e-2d7c1f3e6a10",
  "outboxId": "2b3c4d5e-6f7a-4b8c-9d0e-1f2a3b4c5d6e",
  "status": "PENDING"
}
```
Status: `202`, `401`, `403` (`FORBIDDEN`, role ≠ ADMIN), `404` (`WEBHOOK_DEAD_LETTER_NOT_FOUND`), `409` (`WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`).

### 6.2 Contrato de entrega (nós → cliente)

```http
POST {url cadastrada}
Content-Type: application/json
X-Event-Id: 4f1c2b7e-8a0d-4c55-9b9e-2d7c1f3e6a10
X-Webhook-Id: 9d2f6c1a-3b4e-4f5a-8c7d-1e2f3a4b5c6d
X-Timestamp: 2026-11-20T14:03:12.950Z
X-Signature: sha256=5d41402abc4b2a76b9719d911017c592...

{ ...payload da seção 5.1... }
```

| Header | Conteúdo |
|---|---|
| `X-Event-Id` | UUID do evento, estável entre tentativas e replay. Use-o para deduplicar. |
| `X-Webhook-Id` | id do endpoint cadastrado, para quem tem mais de um. |
| `X-Timestamp` | momento **do envio** (muda a cada tentativa), para o cliente detectar replay attack se quiser. |
| `X-Signature` | `sha256=` + hex de `HMAC-SHA256(secret, rawBody)`. |

- **Durante o grace period da rotação**, o header leva as duas assinaturas: `X-Signature: sha256=<com secret nova>,sha256=<com secret antiga>`. O cliente aceita se qualquer uma bater. (detalhe de implementação)
- O cliente deve responder **2xx em até 10s**. Qualquer outra coisa entra em retry.
- O conteúdo do corpo da resposta do cliente é ignorado, a não ser para o histórico.

---

## 7. Matriz de erros

Todas as classes estendem `AppError` (`src/shared/errors/app-error.ts`) e ficam em `src/modules/webhooks/webhook.errors.ts`, exportadas também por `src/shared/errors/index.ts`.

| Código | HTTP | Onde | Quando | Ação |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | API | `:id` inexistente ou removido | — |
| `WEBHOOK_INVALID_URL` | 400 | API | URL malformada ou com esquema `http` | Cliente corrige a URL |
| `WEBHOOK_INVALID_EVENTS` | 400 | API | Lista vazia ou status fora do enum `OrderStatus` | — |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | API | `customerId` inexistente no cadastro | — |
| `WEBHOOK_INACTIVE` | 409 / — | API e worker | Rotação em webhook inativo; no worker, endpoint inativo na hora do envio | Worker: DLQ sem retry |
| `WEBHOOK_SECRET_REQUIRED` | — | Worker | Endpoint sem secret válida no momento de assinar (estado inconsistente) | DLQ sem retry, log `error` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — | Worker | Corpo > 64 KB | DLQ sem retry, log `error` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | Worker | Sem resposta em 10s | Retry |
| `WEBHOOK_DELIVERY_FAILED` | — | Worker | Status não-2xx, erro de rede, DNS ou TLS | Retry |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | Worker | Falha após a 5ª retentativa | DLQ |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | API admin | Id de DLQ inexistente | — |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | API admin | Registro já reprocessado | — |

Erros do worker não viram resposta HTTP: são gravados em `WebhookDelivery.errorCode`, `WebhookOutbox.lastError` ou `WebhookDeadLetter.reason`.

## 8. Estratégias de resiliência

| Mecanismo | Valor | Origem |
|---|---|---|
| Timeout do POST | 10s | [09:42] Diego |
| Retentativas | 5, em 1m / 5m / 30m / 2h / 12h | [09:17] Larissa |
| Falha permanente | DLQ em tabela separada + replay manual | [09:18] Diego |
| Polling | 2s, batch pequeno em ordem de `created_at` | [09:09] Diego |
| Isolamento | Worker em processo separado; queda do cliente não afeta `changeStatus` | [09:11] Diego |
| Atomicidade | Outbox na mesma transação; falha na inserção faz rollback | [09:40] Bruno |
| Recuperação de órfãos | `PROCESSING` há mais de 60s volta para `PENDING` (worker caiu no meio) | (detalhe de implementação) |
| Shutdown gracioso | `SIGINT`/`SIGTERM` terminam o evento corrente e fazem `prisma.$disconnect()`, como em `src/server.ts` | (detalhe de implementação) |
| Fallback para o cliente | Não há aviso por email nesta fase. Falhas ficam visíveis em `GET /webhooks/:id/deliveries` | [09:37] Larissa |

**Limitação conhecida de ordenação:** com single-worker, os eventos de um pedido saem em ordem de `created_at`. Porém, se o evento A de um pedido estiver aguardando retry, o evento B do mesmo pedido pode ser entregue antes. O cliente deve usar `timestamp` e `to_status` do payload para reconciliar. Não há garantia de ordenação global ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 9. Observabilidade

**Logs (Pino, `src/shared/logger/index.ts`)**, estruturados e sem biblioteca nova:

| Evento | Nível | Campos |
|---|---|---|
| `webhook_event_enqueued` | info | `eventId`, `webhookId`, `orderId`, `toStatus`, `requestId` |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs` |
| `webhook_delivery_failed` | warn | idem + `errorCode`, `nextAttemptAt` |
| `webhook_moved_to_dlq` | warn | `eventId`, `webhookId`, `reason`, `attempts` |
| `webhook_dlq_replayed` | info | `deadLetterId`, `eventId`, `userId` (**auditoria**) |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `signal` |

Adicionar `*.secret` e `*.previousSecret` em `redactPaths` para que a secret nunca apareça em log.

**Métricas.** O projeto ainda não tem stack de métricas, então nesta fase os números saem de consultas no banco e da agregação dos logs. Os nomes abaixo ficam padronizados para quando houver instrumentação. (detalhe de implementação)

| Métrica | Tipo | Para quê |
|---|---|---|
| `webhook_delivery_latency_seconds` | histograma | Do `createdAt` da outbox até o 2xx. Valida o < 10s (OT-1) |
| `webhook_outbox_pending` | gauge | Fila acumulando = worker parado ou lento |
| `webhook_delivery_attempts_total{result}` | contador | Taxa de sucesso/falha por tentativa |
| `webhook_dead_letter_total` | contador | Eventos perdidos para a DLQ |

**Tracing / correlação.** Não há tracing distribuído no projeto. A correlação é feita por ids:
- o `requestId` do `PATCH /orders/:id/status` (gerado em `src/middlewares/request-logger.middleware.ts`) é gravado em `webhook_outbox.requestId`;
- o `eventId` aparece em todos os logs do worker e no header `X-Event-Id`, então dá para seguir um evento da API até o cliente e pedir ao cliente o log pelo mesmo id.

## 10. Dependências e compatibilidade

- **Sem dependências de infraestrutura novas:** mesmo MySQL e mesma `DATABASE_URL`.
- **Bibliotecas:** HMAC e geração de secret com `node:crypto` (`createHmac`, `randomBytes`). UUID com o pacote `uuid` já presente. HTTP com `fetch` nativo do Node ≥ 20 (`engines` do `package.json`) e `AbortSignal.timeout(10_000)`. (detalhe de implementação)
- **Migração Prisma:** só adiciona tabelas e enum, sem mudança em tabelas existentes além da relação reversa em `Customer`.
- **API:** só rotas novas. Nenhum contrato existente muda.
- **Comportamento novo em rota existente:** `PATCH /orders/:id/status` pode falhar com 500 se a inserção na outbox falhar. Isso é intencional (consistência) e deve ser raro.
- **Deploy:** novo processo `npm run worker`, que precisa subir junto com a API.

## 11. Integração com o sistema existente

| Arquivo | Mudança |
|---|---|
| `src/modules/orders/order.service.ts` | Em `changeStatus`, depois de `tx.orderStatusHistory.create(...)`, chamar `publishWebhookEvent(tx, order, from, to, { requestId })`. Usa o `order` já carregado no início da transação (`orderNumber`, `customerId`, `totalCents`). Nenhuma outra lógica do método muda. Como o `OrderService` hoje não recebe `requestId`, o controller passa a repassá-lo (`req.id`). |
| `src/modules/orders/order.status.ts` | Sem mudança. O enum `OrderStatus` validado aqui é a fonte dos valores aceitos no filtro `events`. |
| `prisma/schema.prisma` | Novos models da seção 4 e relação `webhooks WebhookEndpoint[]` em `Customer`. Migration nova em `prisma/migrations/`. |
| `src/shared/errors/app-error.ts` | Base das novas classes (`WebhookNotFoundError`, `WebhookInvalidUrlError` etc.), que passam `statusCode` e `errorCode` `WEBHOOK_*`. Não dá para reaproveitar `NotFoundError` direto porque o código dele é fixo em `NOT_FOUND`. |
| `src/shared/errors/index.ts` | Reexportar os erros do módulo, como já é feito com `InsufficientStockError`. |
| `src/middlewares/error.middleware.ts` | **Sem mudança.** O ramo `err instanceof AppError` já serializa os novos erros. |
| `src/middlewares/auth.middleware.ts` | Reuso de `authenticate` em todas as rotas e de `requireRole('ADMIN')` no replay, no mesmo padrão de `src/modules/users/user.routes.ts`. |
| `src/middlewares/validate.middleware.ts` | Reuso com os schemas de `webhook.schemas.ts`, incluindo `z.string().url().refine(u => u.startsWith('https://'))` para a URL. |
| `src/routes/index.ts` | Registrar `router.use('/webhooks', ...)` e `router.use('/admin/webhooks', ...)` e incluir `webhooks` no tipo `Controllers`. |
| `src/app.ts` | Instanciar `WebhookRepository` → `WebhookService` → `WebhookController` em `buildControllers`. |
| `src/server.ts` | Modelo para o novo `src/worker.ts`: bootstrap, `logger.info` de start e shutdown em `SIGINT`/`SIGTERM` com `prisma.$disconnect()`. |
| `src/config/database.ts` | O worker usa `createPrismaClient()` para ter a própria instância (PrismaClient é por processo). |
| `src/shared/logger/index.ts` | Adicionar paths da secret em `redactPaths`. |
| `package.json` | Scripts `"worker": "tsx --env-file=.env src/worker.ts"` (dev) e equivalente com `node dist/worker.js` para produção. |
| `tests/setup.ts` | Limpar as tabelas novas no `beforeEach`, antes de `order`/`customer` por causa das FKs. |

Estrutura do módulo novo:

```
src/modules/webhooks/
├── webhook.controller.ts
├── webhook.service.ts        # CRUD, rotação, deliveries, replay
├── webhook.repository.ts
├── webhook.routes.ts         # /webhooks e /admin/webhooks
├── webhook.schemas.ts        # Zod
├── webhook.errors.ts         # classes WEBHOOK_*
├── webhook.publisher.ts      # publishWebhookEvent(tx, ...)
├── webhook.signer.ts         # HMAC-SHA256 e geração de secret
└── webhook.worker.ts         # loop de polling, envio, retry, DLQ
src/worker.ts                 # entry-point do processo
```

## 12. Critérios de aceite técnicos

1. `PATCH /orders/:id/status` para um status assinado cria exatamente uma linha `PENDING` por webhook ativo interessado, na mesma transação.
2. Se nenhum webhook ativo do customer assina o status, nenhuma linha é criada.
3. Se a inserção na outbox falhar (simulada em teste), o status do pedido, o histórico e o estoque **não** mudam.
4. Com cliente respondendo 2xx, a entrega ocorre em menos de 10s e a linha fica `DELIVERED`.
5. Com cliente respondendo 500, as tentativas seguem 1m/5m/30m/2h/12h (testado com relógio controlado) e, na falha da 6ª tentativa, o evento vai para `webhook_dead_letter` com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED`.
6. Cliente sem resposta por mais de 10s gera `WEBHOOK_DELIVERY_TIMEOUT` e entra em retry.
7. `X-Signature` confere com `HMAC-SHA256(secret, rawBody)` calculado no teste. Durante o grace period, vêm as duas assinaturas.
8. Uma segunda tentativa do mesmo evento mantém o `X-Event-Id`. O replay da DLQ também.
9. Cadastro com URL `http://` retorna 400 `WEBHOOK_INVALID_URL`.
10. Replay com usuário `OPERATOR` retorna 403. Com `ADMIN`, retorna 202 e gera log `webhook_dlq_replayed` com `userId`.
11. `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, do mais recente para o mais antigo.
12. Nenhum log contém a secret.
13. O worker sobe com `npm run worker`, independente da API, e encerra limpo em `SIGTERM`.

Os testes seguem o padrão de `tests/orders.test.ts` (Vitest + Supertest, factories em `tests/helpers/factories.ts`). O worker é testado chamando a função de processamento de um ciclo diretamente, com um servidor HTTP local fazendo papel de cliente.

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Worker parado acumula eventos e estoura os 10s | Média | Alto | Monitorar `webhook_outbox_pending`. Processo supervisionado com restart automático. Recuperação de órfãos `PROCESSING` |
| Crescimento contínuo da outbox e do histórico de entregas | Alta | Médio | Índices em `status`/`created_at`. Arquivamento (cerca de 30 dias) como trabalho futuro (fora do escopo) |
| Cliente processa evento duplicado | Média | Médio | `X-Event-Id` estável. Documentação destacada no portal do desenvolvedor |
| Vazamento de secret | Baixa | Alto | Secret por endpoint, rotação com 24h, redaction no logger, revisão de segurança da Sofia antes do deploy |
| Evento fora de ordem durante retry | Média | Baixo | `timestamp` e `from_status`/`to_status` no payload. Limitação documentada |
| Inserção na outbox faz `changeStatus` falhar | Baixa | Alto | Inserção simples e indexada. Teste de rollback. Monitorar 500 em `PATCH /orders/:id/status` |
| Rajada de eventos sobrecarrega o cliente (sem rate limit) | Baixa | Médio | Observar volume por customer. Decisão adiada (ver RFC, questões em aberto) |
