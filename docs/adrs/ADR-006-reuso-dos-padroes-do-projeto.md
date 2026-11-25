# ADR-006: Webhooks como módulo comum, reutilizando os padrões existentes do projeto

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Bruno, Diego

## Contexto

A codebase tem padrões claros e consistentes. A feature de webhooks poderia trazer estrutura própria (outro logger, outro formato de erro, outra organização de pastas), mas isso aumentaria o custo de manutenção e a curva para o time.

Padrões que já existem e são relevantes:

| Padrão | Onde está |
|---|---|
| Módulo por domínio com controller, service, repository, routes e schemas | `src/modules/orders/`, `src/modules/customers/` etc. |
| Erros de aplicação com `statusCode` + `errorCode` | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| Erros específicos com código em caixa alta (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) | `src/shared/errors/http-errors.ts` |
| Tratamento centralizado de `AppError`, `ZodError` e erros do Prisma | `src/middlewares/error.middleware.ts` |
| Logger Pino com redaction | `src/shared/logger/index.ts` |
| Validação com Zod via middleware | `src/middlewares/validate.middleware.ts` |
| Autenticação JWT e controle por role | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| IDs UUID em todas as tabelas | `prisma/schema.prisma` |

## Decisão

Reuso máximo do que já existe:

- Novo módulo **`src/modules/webhooks/`** com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts`, mais o arquivo de processamento do worker (`webhook.worker.ts` ou `webhook.processor.ts`).
- Erros do módulo estendem `AppError` (ou as classes de `http-errors.ts`) e usam **prefixo `WEBHOOK_`** nos códigos (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` etc.).
- Logs via o `logger` Pino existente, **sem biblioteca nova**.
- Nenhuma mudança no `error.middleware.ts`: ele já trata `AppError` e vai pegar os erros do módulo.
- Rotas de CRUD com `authenticate`, e o replay da DLQ com `requireRole('ADMIN')`.
- Tabelas novas com **id UUID**, seguindo o resto do schema.
- A integração com pedidos é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client da transação, em vez de injetar um repository inteiro no `OrderService`.

## Alternativas consideradas

### Estrutura própria para o módulo de webhooks
Organização de pastas, logger ou formato de erro dedicados.
**Descartada:** quebra a consistência da codebase sem ganho concreto.

### Injetar o repository de webhooks no `OrderService`
**Descartada em favor da função pura recebendo `tx`:** acopla menos e deixa explícito que a escrita acontece dentro da transação corrente.

### Id auto incremental na outbox
**Descartada:** tudo no projeto é UUID ([09:51] Larissa).

## Consequências

**Positivas**
- Quem conhece um módulo do projeto entende o de webhooks.
- As respostas de erro saem no mesmo formato JSON (`{ error: { code, message, details } }`) sem trabalho extra.
- Sem dependências novas para logging ou tratamento de erros.

**Negativas / trade-offs**
- Herda as limitações dos padrões atuais. Por exemplo, o CRUD de webhook aceita qualquer role autenticada, igual aos outros módulos, e o endurecimento de permissões fica para depois.
- `OrderService` passa a depender de uma função do módulo de webhooks, criando um acoplamento novo entre os módulos `orders` e `webhooks`.

## Referências

- Transcrição: [09:27]–[09:30] (Bruno, Diego, Larissa), [09:36] (Larissa), [09:41] (Bruno, Diego), [09:51] (Diego, Larissa)
- Código: arquivos listados na tabela acima
