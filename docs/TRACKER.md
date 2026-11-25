# Tracker de Rastreabilidade

Cada item dos documentos ligado à sua origem: a transcrição da reunião (`TRANSCRICAO.md`) ou o código do repositório.
Para a transcrição, a localização é `[hh:mm] Nome` da fala. Para o código, o caminho do arquivo.

## Itens rastreados

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em `GET /orders`, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco de negócio | Atlas pode migrar para concorrente se não sair até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Métrica | "Tempo real" = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Prazo | Atlas quer a feature até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Prazo | Estimativa de três sprints com revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Janela de ~15h coberta pelas retentativas | TRANSCRICAO | [09:17] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook via POST com URL e lista de status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | docs/PRD.md | Restrição | `customerId` vai no body/path, não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status em cada endpoint | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 em cada notificação | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com antiga válida por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | 5 retentativas em 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | DLQ com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay manual da DLQ restrito a ADMIN, com log de quem fez | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| PRD-FR-13b | docs/PRD.md | Requisito Funcional | Header `X-Webhook-Id` para quem tem vários cadastros | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Campos do payload e `event_type` `order.status_changed` | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Worker lê a outbox a cada 2s; atende < 10s | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Status não muda sem evento; rollback leva o evento junto | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não pode travar mudança de status de outros pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once com dedup pelo cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | URL obrigatoriamente https; http recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB, erro em vez de truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s na chamada ao cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Ordenação só por pedido e enquanto single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-10 | docs/PRD.md | Restrição | Sem infra nova; time pequeno | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Auditoria do replay da DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | Email de alerta de falha adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard visual é projeto separado do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Rate limiting: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Apenas outbound; clientes não enviam webhooks | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Arquivamento de entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo | Itens do pedido não vão no payload | TRANSCRICAO | [09:43] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Dois dias úteis de revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Marcos confirma prazo e atualiza os clientes | TRANSCRICAO | [09:49] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Responsabilidade de dedup passa ao cliente | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente já vazou secret em log | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente com indisponibilidade longa (já houve 2h) | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajada de 50 mudanças/min bombardeia o cliente | TRANSCRICAO | [09:38] Diego |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service de orders | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / Redis Cluster | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger no banco para notificar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-04b | docs/RFC.md | Alternativa descartada | Apenas 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers (partição por order_id ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Arquivamento da outbox sem dono nem prazo | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD de webhook | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | Email ao cliente em falhas recorrentes | TRANSCRICAO | [09:37] Marcos |
| RFC-IMP-01 | docs/RFC.md | Impacto | `changeStatus` passa a gravar na outbox dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMP-02 | docs/RFC.md | Impacto | Revisão de segurança incluída no fim das 3 sprints | TRANSCRICAO | [09:47] Larissa |
| FDD-DADOS-01 | docs/FDD.md | Decisão | Ids UUID nas tabelas novas | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-01b | docs/FDD.md | Padrão existente | Todas as tabelas usam `@id @default(uuid()) @db.Char(36)` | CODIGO | prisma/schema.prisma |
| FDD-DADOS-02 | docs/FDD.md | Decisão | Outbox com status pendente/processando/falhou/entregue e índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Decisão | Configuração guarda url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-04 | docs/FDD.md | Decisão | Tabela `webhook_dead_letter` separada | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-05 | docs/FDD.md | Padrão existente | Campos do snapshot vêm do model `Order` (`orderNumber`, `totalCents`, `customerId`) | CODIGO | prisma/schema.prisma |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o tx | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Restrição | Inserção na outbox na mesma transação; falha → rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | Filtro aplicado na inserção; sem interessado, não insere | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-05 | docs/FDD.md | Decisão | Worker lê pendentes mais antigos em batch pequeno a cada 2s | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Decisão | Progressão 1m/5m/30m/2h/12h, ~15h no total | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Decisão | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks` com url, events e customerId | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks` por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Endpoint de rotação de secret pela API | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Padrão existente | Rotas sob o prefixo `/api/v1` | CODIGO | src/app.ts |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Headers de entrega e `X-Timestamp` para detecção de replay attack | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | HMAC calculado sobre o corpo do request | TRANSCRICAO | [09:22] Sofia |
| FDD-CONTRATO-11 | docs/FDD.md | Padrão existente | Formato de erro `{ error: { code, message, details } }` | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERRO-01 | docs/FDD.md | Decisão | Prefixo `WEBHOOK_` em todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Decisão | Exemplos `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Padrão existente | Classes estendem `AppError` (statusCode + errorCode) | CODIGO | src/shared/errors/app-error.ts |
| FDD-ERRO-04 | docs/FDD.md | Padrão existente | Erros específicos como `InsufficientStockError` / `InvalidStatusTransitionError` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-05 | docs/FDD.md | Requisito Não Funcional | `WEBHOOK_PAYLOAD_TOO_LARGE` acima de 64 KB | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-06 | docs/FDD.md | Requisito Não Funcional | `WEBHOOK_DELIVERY_TIMEOUT` após 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-01 | docs/FDD.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| FDD-RES-02 | docs/FDD.md | Limitação | Ordenação por created_at com single-worker; sem ordenação global | TRANSCRICAO | [09:12] Diego |
| FDD-OBS-01 | docs/FDD.md | Decisão | Logs com o Pino já existente, sem lib nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Padrão existente | Logger Pino com `redactPaths` | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Padrão existente | `requestId` gerado por request para correlação | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-04 | docs/FDD.md | Requisito Não Funcional | Log de auditoria de quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-DEP-01 | docs/FDD.md | Decisão | Mesmo banco e DATABASE_URL, PrismaClient separado por processo | TRANSCRICAO | [09:30] Bruno |
| FDD-DEP-02 | docs/FDD.md | Padrão existente | `createPrismaClient()` para nova instância | CODIGO | src/config/database.ts |
| FDD-INT-01 | docs/FDD.md | Integração | Extensão de `changeStatus` (update, history, estoque na mesma tx) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Middleware de erro já trata AppError, Zod e Prisma | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-03 | docs/FDD.md | Integração | Reuso de `requireRole('ADMIN')` no replay | TRANSCRICAO | [09:36] Larissa |
| FDD-INT-04 | docs/FDD.md | Integração | `authenticate` / `requireRole` existentes | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Registro dos novos routers e tipo `Controllers` | CODIGO | src/routes/index.ts |
| FDD-INT-06 | docs/FDD.md | Integração | `src/worker.ts` no molde de `src/server.ts` + `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-07 | docs/FDD.md | Integração | Bootstrap e shutdown em SIGINT/SIGTERM usados como modelo | CODIGO | src/server.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Validação Zod via `validate` (https no schema) | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Enum `OrderStatus` e transições válidas | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Limpeza das tabelas novas nos testes | CODIGO | tests/setup.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Módulo em `src/modules/webhooks` com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-12 | docs/FDD.md | Integração | Lógica do worker em `webhook.worker.ts` / `webhook.processor.ts` | TRANSCRICAO | [09:28] Bruno |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em polling de 2s, processo separado | TRANSCRICAO | [09:10] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 tentativas com backoff e DLQ separada | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-006b | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Padrão existente | Estrutura de módulo controller/service/repository/routes/schemas | CODIGO | src/modules/orders/order.routes.ts |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Bruno |
| ADR-007b | docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md | Decisão | Payload enxuto, sem itens | TRANSCRICAO | [09:44] Bruno |

**Totais:** 120 linhas · 102 com fonte TRANSCRICAO (85%) · 18 com fonte CODIGO. Todos os timestamps e falantes conferidos contra a transcrição e todos os caminhos de código conferidos contra o repositório.

Arquivos citados nos documentos que **ainda não existem** são os novos, propostos pela feature (`src/worker.ts` e `src/modules/webhooks/*`). Nenhum deles aparece como se já existisse.

## Itens sem origem direta (detalhes de implementação propostos no FDD)

Estes pontos não foram discutidos na reunião. São escolhas de implementação necessárias para o FDD ficar acionável e estão marcados como "(detalhe de implementação)" no próprio documento. Ficam listados aqui para revisão explícita do time na sessão de design citada pela Larissa ([09:50] Larissa).

| Item | Onde | Por que foi necessário |
|---|---|---|
| Uma linha de outbox por endpoint de destino | FDD §4 | Cada endpoint tem secret e `X-Webhook-Id` próprios |
| Remoção lógica de webhook (`active = false`) | FDD §4, §6.1 | Preservar histórico de entregas e DLQ |
| Tabela `webhook_deliveries` | FDD §4 | Sustentar o requisito do histórico ([09:34] Marcos) |
| Recuperação de linhas `PROCESSING` órfãs após 60s | FDD §5.2, §8 | Worker pode cair no meio do envio |
| Batch de 20 eventos por ciclo | FDD §5.2 | "Batch pequeno" ([09:08] Diego) precisava de um número |
| Duas assinaturas no `X-Signature` durante a rotação | FDD §6.2 | Tornar operável o grace period de 24h |
| Corpo de resposta truncado em 4 KB no histórico | FDD §4 | Evitar crescimento descontrolado da tabela |
| Nomes de métricas | FDD §9 | O projeto ainda não tem stack de métricas |
| `fetch` nativo + `node:crypto` | FDD §10 | Evitar dependências novas, em linha com [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
