# ADR-002: Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e entregá-los. O requisito de produto é que o cliente receba a notificação em **menos de 10 segundos** ([09:02] Marcos).

Duas perguntas ficaram em aberto: como o worker descobre que há evento novo e onde ele roda.

## Decisão

1. **Polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos em batch pequeno, processando e marcando o resultado.
2. **Processo separado da API**, com entry-point próprio `src/worker.ts` (no mesmo estilo de `src/server.ts`) e script `npm run worker`. A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`).
3. Mesmo banco e mesma `DATABASE_URL`, mas com **instância própria de `PrismaClient`**, porque o client é por processo.
4. **Um único worker** nesta fase. Com isso, os eventos são processados em ordem de `created_at` e a ordenação fica garantida por pedido.

## Alternativas consideradas

### Trigger no banco para ser mais reativo
**Descartada:** o MySQL não tem um mecanismo tipo `NOTIFY/LISTEN` do Postgres. A trigger só executa SQL e não avisa processo externo; avisar o worker exigiria gambiarra (escrever em arquivo, chamar endpoint).

### Worker rodando dentro do processo da API
**Descartada:** se a API reinicia, o worker cai junto. Processo separado isola os ciclos de vida.

## Consequências

**Positivas**
- Simples de implementar e operar, sem dependência nova.
- Latência de pior caso de cerca de 2s até o primeiro envio, dentro do limite de 10s.
- Ordenação por `order_id` enquanto houver um só worker.

**Negativas / trade-offs**
- Há latência mínima de até 2s, aceita explicitamente ([09:10] Larissa).
- Consultas periódicas no banco mesmo sem eventos.
- **Não há ordenação global** e, se no futuro houver múltiplos workers em paralelo, perde-se também a ordenação por pedido. Escalar exigiria particionar por `order_id` ou lock pessimista. Isso foi registrado como limitação conhecida e problema futuro.

## Referências

- Transcrição: [09:09]–[09:14] (Diego, Bruno, Larissa, Marcos), [09:28] (Bruno), [09:29]–[09:30] (Diego, Bruno)
- Código: `src/server.ts`, `src/config/database.ts` (`createPrismaClient`), `package.json` (scripts)
