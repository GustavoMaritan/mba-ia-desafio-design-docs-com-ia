# PRD — Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Produto** | Order Management System (OMS) |
| **Responsável de produto** | Marcos (PM) |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião técnica, aguardando implementação |
| **Documentos técnicos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) |

## 1. Resumo e contexto

Vamos permitir que clientes B2B recebam uma **notificação HTTP (webhook) sempre que o status de um pedido deles mudar**, em vez de ficarem consultando a nossa API repetidamente. O cliente cadastra uma URL, escolhe quais status quer acompanhar e passa a receber chamadas assinadas que provam que vieram da nossa plataforma.

## 2. Problema e motivação

- Atlas Comercial, MaxDistribuição e Nova Cargo fizeram **pedido formal** desta funcionalidade.
- Hoje eles fazem polling em `GET /orders` de tempos em tempos para detectar mudanças, o que deixa a integração **lenta e cara** para eles.
- A Atlas sinalizou que pode **migrar para um concorrente** se a entrega não acontecer até o fim do trimestre. Isso transforma a feature em risco de receita.
- O OMS não tem nenhum mecanismo de notificação externa hoje.

## 3. Público-alvo e cenários de uso

**Público:** clientes B2B que integram sistemas próprios (ERP, logística, atendimento) com o nosso OMS, inicialmente os três solicitantes. A configuração é feita pela nossa API, com usuários autenticados que representam o cliente.

**Cenários**

1. **Só o que interessa:** um cliente assina apenas `SHIPPED` e `DELIVERED` e deixa de fazer polling em `GET /orders` para saber quando o pedido saiu e quando chegou.
2. **Detalhes sob demanda:** o cliente recebe o evento enxuto e, se precisar dos itens, consulta `GET /orders/:id`.
3. **Cliente com mais de um cadastro:** o cliente cadastra mais de um endpoint e usa o `X-Webhook-Id` para saber qual cadastro gerou cada chamada.
4. **Cliente fora do ar em manutenção:** o endpoint do cliente cai por 2h. As notificações são reenviadas automaticamente quando ele volta, sem ação de ninguém.
5. **Secret vazada:** o cliente percebe que a secret apareceu num log, pede rotação e tem 24h para atualizar os sistemas antes de a antiga deixar de valer.
6. **Investigação de falha:** o cliente consulta o histórico das últimas entregas para ver o que falhou e o que o servidor dele respondeu.

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Notificação percebida como "tempo real" | Tempo entre a mudança de status e a entrega ao cliente (em condições normais) | **< 10 segundos** |
| Atender os clientes que pediram | Clientes solicitantes integrados via webhook | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) |
| Cumprir o prazo comercial | Feature em produção | Até o **fim de novembro**, em **3 sprints** incluindo a revisão de segurança |
| Resiliência a indisponibilidade do cliente | Janela coberta por reenvio automático antes de exigir ação manual | **~15 horas** (5 retentativas) |

## 5. Escopo

### 5.1 Incluso
- Webhooks **de saída** (nós → cliente) para mudança de status de pedido.
- Cadastro, listagem, edição e remoção de endpoints de webhook por customer.
- Filtro por status em cada endpoint.
- Secret única por endpoint, gerada pela plataforma, com rotação.
- Reenvio automático em caso de falha e fila de eventos que falharam definitivamente (DLQ).
- Reprocessamento manual da DLQ por administrador.
- Histórico das últimas entregas por endpoint.

### 5.2 Fora de escopo

| Item | Situação | Origem |
|---|---|---|
| **Email avisando o cliente** quando o webhook dele falha repetidamente | Adiado para a próxima fase, depois de medir impacto | [09:37] Larissa |
| **Dashboard visual** para o cliente gerenciar webhooks | Descartado nesta feature. Projeto separado do time de frontend | [09:40] Larissa |
| **Rate limiting** de envio por cliente | Não entra. Observar e decidir depois | [09:39] Diego, Larissa |
| **Webhooks de entrada** (cliente enviando para nós) | Não faz parte. Clientes só querem receber | [09:02] Marcos |
| **Arquivamento** de eventos entregues (cerca de 30 dias) | Fora do escopo desta feature | [09:08] Diego |
| **Ordenação global** de eventos e múltiplos workers | Limitação conhecida. Ordenação apenas por pedido | [09:13] Larissa |
| **Itens do pedido no payload** | Não enviados. O cliente consulta `GET /orders/:id` se precisar | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito |
|---|---|
| RF-01 | O cliente cadastra um webhook informando `url`, `customerId` e a lista de status que quer receber. O `customerId` é informado na requisição e **não** é derivado do JWT. |
| RF-02 | A plataforma gera a secret do webhook e a devolve na resposta do cadastro. |
| RF-03 | O cliente lista os webhooks de um customer. |
| RF-04 | O cliente edita um webhook (URL, status assinados, ativo/inativo). |
| RF-05 | O cliente remove um webhook. |
| RF-06 | Quando o status de um pedido muda, o sistema notifica todos os webhooks ativos do customer que assinam o novo status. Webhooks que não assinam aquele status não recebem nada. |
| RF-07 | Cada notificação é assinada com HMAC-SHA256 usando a secret do endpoint. |
| RF-08 | O cliente pode rotacionar a secret. A secret anterior continua válida por 24 horas. |
| RF-09 | Notificações que falham são reenviadas automaticamente até 5 vezes, com intervalos crescentes (1 min, 5 min, 30 min, 2 h, 12 h). |
| RF-10 | Esgotadas as tentativas, o evento vai para uma fila de falhas (DLQ) com o payload, o motivo e a data. |
| RF-11 | Um usuário com role **ADMIN** pode reprocessar manualmente um evento da DLQ. O sistema registra quem fez o reprocessamento. |
| RF-12 | O cliente consulta o histórico das últimas 100 entregas de um webhook, com sucesso/falha, payload, resposta e tempo de resposta. |
| RF-13 | Cada notificação leva identificador único do evento (`X-Event-Id`), identificador do webhook (`X-Webhook-Id`) e timestamp do envio (`X-Timestamp`). |
| RF-14 | O payload contém `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, refletindo o pedido no momento da mudança. |

## 7. Requisitos não funcionais

| ID | Requisito |
|---|---|
| RNF-01 | **Latência:** entrega em menos de 10s. O ciclo de leitura do worker é de 2s. |
| RNF-02 | **Consistência:** toda mudança de status commitada que tenha webhook interessado gera evento, e nenhuma mudança desfeita gera evento. |
| RNF-03 | **Isolamento:** a lentidão ou indisponibilidade de um cliente não afeta a mudança de status de nenhum pedido. |
| RNF-04 | **Entrega at-least-once:** o cliente pode receber duplicatas e deve deduplicar pelo `X-Event-Id`. |
| RNF-05 | **Segurança de transporte:** URLs de webhook precisam ser `https`. Cadastro com `http` é recusado. |
| RNF-06 | **Segurança de segredo:** secret única por endpoint, nunca global e nunca exposta em log. |
| RNF-07 | **Tamanho:** eventos acima de 64 KB não são enviados (erro, sem truncar). |
| RNF-08 | **Timeout:** o cliente tem 10s para responder. Depois disso, conta como falha. |
| RNF-09 | **Ordenação:** garantida apenas por pedido, enquanto houver um único worker. |
| RNF-10 | **Operação:** sem infraestrutura nova. Usa o MySQL existente e um processo worker separado da API. |
| RNF-11 | **Auditoria:** o reprocessamento manual da DLQ fica registrado com o usuário responsável. |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Evento gravado numa outbox no MySQL, na mesma transação da mudança de status | Banco existente vira também a fila. Tabela cresce e o arquivamento fica para depois | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado em polling de 2s | Latência mínima de até 2s e consultas periódicas no banco | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15h e depois DLQ | Cliente fora por mais de ~15h depende de replay manual | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| HMAC-SHA256 com secret por endpoint e rotação de 24h | Mais complexidade no modelo e no envio durante a rotação | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Cliente precisa deduplicar | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Herda limitações atuais, como CRUD aberto a qualquer role autenticada | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload em snapshot e filtro na inserção | Mudança de filtro não afeta eventos já gerados | [ADR-007](adrs/ADR-007-payload-snapshot-e-filtro-na-insercao.md) |

## 9. Dependências

- **Revisão de segurança da Sofia:** pelo menos dois dias úteis antes do deploy, focada em HMAC e geração de secret.
- **Portal do desenvolvedor (Marcos):** documentar a integração, a verificação de assinatura e, com destaque, a deduplicação por `X-Event-Id`.
- **Infra de deploy:** subir e supervisionar um novo processo (worker) além da API.
- **Comunicação com os clientes:** Marcos confirma prazo e atualiza os três clientes.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Atraso na entrega e perda da Atlas para concorrente | Média | Alto | Escopo enxuto (sem dashboard, email ou rate limit). Estimativa de 3 sprints já com revisão de segurança. Prazo confirmado com o cliente |
| Cliente processa notificação duplicada e gera efeito colateral no sistema dele | Média | Médio | `X-Event-Id` estável entre tentativas. Documentação destacada no portal |
| Vazamento de secret de um cliente | Baixa | Alto | Secret por endpoint (vazamento não se espalha), rotação com 24h de transição, revisão de segurança |
| Cliente fica fora do ar por mais de ~15h e perde eventos | Baixa | Médio | DLQ com replay manual por admin. Histórico de entregas visível ao cliente |
| Rajada de mudanças de status sobrecarrega o endpoint do cliente | Baixa | Médio | Monitorar volume. Rate limiting fica como decisão futura |
| Worker parado deixa notificações atrasadas | Média | Alto | Processo separado e supervisionado. Monitoramento de eventos pendentes |

## 11. Critérios de aceitação

1. Um cliente consegue cadastrar, listar, editar e remover webhooks pela API, e recebe a secret no cadastro.
2. Um cadastro com URL `http://` é recusado com erro de validação.
3. Ao mudar o status de um pedido para um status assinado, o endpoint do cliente recebe a notificação em menos de 10s.
4. Ao mudar para um status **não** assinado, nada é enviado.
5. A assinatura `X-Signature` pode ser validada pelo cliente com a secret recebida.
6. Após rotação, notificações continuam válidas com a secret antiga por 24h e depois só com a nova.
7. Com o endpoint do cliente fora do ar, o sistema reenvia nos intervalos definidos e, ao esgotar, o evento aparece na DLQ.
8. Um ADMIN reprocessa um evento da DLQ e o cliente o recebe com o mesmo `X-Event-Id`. Um OPERATOR não consegue reprocessar.
9. O histórico de entregas mostra as últimas tentativas com resultado e tempo de resposta.
10. Uma mudança de status que falha (por exemplo, transição inválida ou estoque insuficiente) não gera notificação.

## 12. Estratégia de testes e validação

- **Testes de integração** no padrão já usado no projeto (Vitest + Supertest), cobrindo o CRUD, as validações e a geração do evento junto com a mudança de status, inclusive o rollback quando algo falha.
- **Testes do worker** com um servidor HTTP local simulando o cliente: sucesso, erro 500, timeout, payload grande, sequência de retentativas e ida para a DLQ.
- **Testes de segurança:** verificação da assinatura HMAC, comportamento durante a rotação e ausência da secret nos logs, revisados pela Sofia antes do deploy.
- **Validação da meta de latência:** medir o tempo entre a mudança de status e a resposta 2xx do cliente e comparar com a meta de 10s.
