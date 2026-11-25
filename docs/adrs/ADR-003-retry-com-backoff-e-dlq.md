# ADR-003: Retry com backoff exponencial fixo e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Larissa, Diego, Bruno

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou respondendo erro. Já houve cliente com **duas horas de indisponibilidade** em manutenção planejada ([09:16] Diego). Precisamos tentar de novo sem segurar eventos para sempre e sem desistir cedo demais.

## Decisão

- **Backoff com progressão fixa:** após a primeira falha, novas tentativas em **1 min, 5 min, 30 min, 2 h e 12 h**, ou seja, 5 retentativas cobrindo quase 15 horas entre a primeira falha e a última tentativa.
- Esgotadas as tentativas, o evento é considerado **falha permanente** e vai para a **DLQ**.
- A DLQ é uma **tabela separada `webhook_dead_letter`**, com o payload, o motivo da falha e o timestamp.
- O reprocessamento é **manual**, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Esse endpoint exige role `ADMIN` e registra em log quem fez o replay.
- Timeout de 10 segundos sem resposta também conta como falha e entra no ciclo de retry.

## Alternativas consideradas

### Retry indefinido com backoff
**Descartada:** um evento cujo cliente "sumiu" ficaria pendurado para sempre.

### 3 tentativas
**Descartada:** é pouco. Uma indisponibilidade de manhã esgotaria as tentativas em cerca de 30 minutos, e não cobre o caso real de 2h de manutenção.

### Marcar como `failed` na própria outbox (sem tabela de DLQ)
**Descartada:** a tabela separada deixa a leitura da outbox principal mais limpa e funciona como evidência para debug e reprocessamento.

## Consequências

**Positivas**
- Cobre indisponibilidades longas (até cerca de 15h) sem intervenção.
- Falhas permanentes ficam isoladas e auditáveis.
- O replay é controlado e rastreável (role ADMIN + log de quem fez).

**Negativas / trade-offs**
- Um cliente fora do ar por mais de cerca de 15h perde eventos até alguém fazer replay manual. Isso foi aceito pelo produto ([09:17] Marcos).
- Não há aviso automático ao cliente sobre falhas: o fallback por email ficou para a próxima fase.
- O replay manual depende de ação operacional.

## Referências

- Transcrição: [09:14]–[09:18] (Larissa, Diego, Bruno, Marcos), [09:35]–[09:36] (Diego, Larissa, Sofia), [09:42] (Diego)
- Código: `src/middlewares/auth.middleware.ts` (`requireRole`)
