# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

Entrega do desafio **Design Docs com IA** do MBA em Engenharia de Software com IA (Full Cycle).
O enunciado original está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

O ponto de partida é uma reunião de 55 minutos entre tech lead, PM, dois engenheiros e a engenheira de segurança. Nela, o time decide como construir um sistema de webhooks para avisar clientes B2B quando o status de um pedido muda num OMS em Node.js/TypeScript com Prisma e MySQL. Ninguém registrou nada além da transcrição.

O trabalho foi transformar essa conversa, junto com o código existente, num pacote de documentação que um dev consiga pegar e começar a implementar: PRD, RFC, FDD, ADRs e um tracker que liga cada afirmação à fala ou ao arquivo de onde ela saiu. A parte mais difícil não foi escrever, e sim separar o que foi decidido do que foi só mencionado, descartado ou adiado.

## Ferramentas de IA utilizadas

- **Claude Code (CLI):** ferramenta principal. Leu o repositório inteiro e a transcrição, gerou os rascunhos de cada documento e fez a validação cruzada do tracker. Trabalhar direto no repositório ajudou muito: a IA conseguia abrir `order.service.ts` ou `error.middleware.ts` em vez de supor como eles funcionavam.
- **Script Python de validação:** pedi para a IA gerar um script curto que confere se cada linha do tracker aponta para um timestamp + falante que realmente existe na transcrição e para um arquivo que existe no repositório. Ele rodou a cada revisão.

## Workflow adotado

Segui a ordem sugerida no enunciado, com uma etapa de extração antes:

1. **Contextualização:** leitura do código (`src/modules/orders`, `src/shared/errors`, middlewares, `prisma/schema.prisma`, `src/server.ts`) e da transcrição completa.
2. **Extração estruturada da reunião:** antes de escrever qualquer documento, pedi uma lista classificada de tudo que aparece na transcrição (decisão, requisito, restrição, descartado, adiado, em aberto), sempre com o timestamp. Essa lista virou a base de tudo e depois a espinha do tracker.
3. **ADRs:** uma decisão por arquivo, começando pelas 6 principais. Separei o snapshot do payload e o filtro na inserção num ADR-007, porque foram decididos depois do resumo da Larissa e mudam a modelagem da outbox.
4. **RFC:** curto, só com abordagem, alternativas e pontos em aberto, linkando os ADRs.
5. **FDD:** modelo de dados, fluxos, contratos, matriz de erros e a seção de integração com o código real.
6. **PRD:** por último entre os grandes, consolidando o que já estava decidido em linguagem de produto.
7. **Tracker:** montado varrendo os documentos prontos e validado com o script.
8. **Revisão contra a checklist do enunciado**, item por item.

## Prompts customizados

**Extração classificada da transcrição.** Esse prompt foi o que mais fez diferença. Pedidos genéricos tipo "resuma a reunião" misturavam o que foi descartado com o que foi decidido.

```text
Leia TRANSCRICAO.md inteira. Não resuma.
Liste cada ponto relevante da reunião numa tabela com as colunas:
timestamp | falante | tipo | conteúdo em uma linha
onde tipo é exatamente um de: DECISAO, REQUISITO_FUNCIONAL, REQUISITO_NAO_FUNCIONAL,
RESTRICAO, DESCARTADO, ADIADO, EM_ABERTO, DETALHE_TECNICO.

Regras:
- Use o timestamp e o nome de quem FALOU, não de quem perguntou.
- Se algo foi proposto e depois descartado, registre as duas falas.
- "Fica pra próxima fase", "observar e decidir depois" e "fora do escopo" são ADIADO ou EM_ABERTO, nunca REQUISITO.
- Não complete lacunas. Se a reunião não definiu um número, não invente um.
```

**FDD amarrado ao código real.**

```text
Escreva a seção "Integração com o sistema existente" do docs/FDD.md.
Para cada arquivo, ABRA o arquivo antes de escrever sobre ele e cite o nome real
de funções e classes (ex.: changeStatus, requireRole, AppError, redactPaths).
Cubra pelo menos: order.service.ts, schema.prisma, app-error.ts, error.middleware.ts,
auth.middleware.ts, routes/index.ts, server.ts, database.ts.
Diga o que muda e, quando for o caso, diga explicitamente "sem mudança" e por quê.
Não cite nenhum caminho que não exista, exceto os arquivos novos do módulo webhooks,
e deixe claro quando o arquivo é novo.
Tudo que for escolha de implementação não discutida na reunião deve ser marcado
como "(detalhe de implementação)".
```

**Caça a invenções no PRD** (usado na revisão):

```text
Releia docs/PRD.md e, para cada frase que afirma um fato, requisito, métrica ou cenário,
aponte o timestamp da transcrição ou o arquivo de código que a sustenta.
Liste separadamente as frases SEM origem. Não corrija ainda, só liste.
```

## Iterações e ajustes

Foram **quatro ciclos principais** de geração e revisão. Os ajustes mais relevantes:

1. **Contagem de tentativas do retry.** "5 tentativas" é ambíguo: podem ser 5 envios no total ou 5 retentativas depois do primeiro envio. Só que a progressão 1m/5m/30m/2h/12h tem 5 intervalos, e o Diego fala em "quase 15 horas entre primeira falha e última tentativa", o que só fecha com o envio inicial mais 5 retentativas (14h36). Deixei isso explícito no ADR-003, no FDD e no PRD, com uma tabela de tentativas no FDD para não deixar dúvida.
2. **Onde validar o limite de 64 KB.** O caminho óbvio seria validar o tamanho do payload na hora de inserir na outbox, já que o snapshot é montado ali. Isso faria a mudança de status do pedido falhar por causa de um webhook, exatamente o acoplamento que a reunião quis evitar. Movi a checagem para o worker: o evento vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry e sem afetar o `changeStatus`.
3. **Cenários e validações inventados no PRD.** O primeiro PRD atribuía cenários específicos a cada cliente ("a Nova Cargo assina SHIPPED…") e propunha uma "validação com cliente piloto". Nada disso estava na reunião. O prompt de caça a invenções pegou os dois: reescrevi os cenários de forma genérica, só com o que foi dito, e troquei o piloto pela medição da meta de 10s.
4. **Ordenação durante retry.** A IA repetiu "ordenação garantida por pedido com single-worker", mas não percebeu que, se o evento A de um pedido estiver esperando retry, o evento B do mesmo pedido pode sair antes. Registrei isso como limitação conhecida no FDD e o risco correspondente.
5. **Detalhes de implementação sem dono.** Para o FDD ficar acionável, algumas coisas precisaram ser definidas sem terem sido discutidas: recuperação de eventos órfãos, tamanho do batch, duas assinaturas durante a rotação, tabela de histórico de entregas. Em vez de deixar passarem como se fossem decisões do time, marquei todas como "(detalhe de implementação)" e listei numa seção própria do tracker para revisão.
6. **Totais do tracker.** O cabeçalho de totais estava com números estimados. O script de validação mostrou a contagem real (120 linhas, 85% transcrição) e corrigi.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](docs/PRD.md): o problema, para quem e o que entra ou não.
2. [`docs/RFC.md`](docs/RFC.md): a proposta técnica em 2 a 4 páginas, as alternativas descartadas e o que ficou em aberto.
3. [`docs/adrs/`](docs/adrs/README.md): sete decisões, uma por arquivo.
   - [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md) Outbox no MySQL
   - [ADR-002](docs/adrs/ADR-002-worker-separado-em-polling.md) Worker separado em polling
   - [ADR-003](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md) Retry com backoff e DLQ
   - [ADR-004](docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) HMAC-SHA256 com secret por endpoint
   - [ADR-005](docs/adrs/ADR-005-at-least-once-com-x-event-id.md) At-least-once com X-Event-Id
   - [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) Reuso dos padrões do projeto
   - [ADR-007](docs/adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md) Snapshot do payload e filtro na inserção
4. [`docs/FDD.md`](docs/FDD.md): como construir, com modelo de dados, fluxos, contratos, erros e integração com o código.
5. [`docs/TRACKER.md`](docs/TRACKER.md): de onde veio cada item.

`TRANSCRICAO.md`, `src/`, `prisma/` e `tests/` não foram alterados.
