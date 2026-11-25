# ADR-004: Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00
- **Decisores:** Sofia (Segurança), Larissa, Diego, Bruno

## Contexto

Os eventos carregam dados de pedidos e saem para endpoints fora da nossa infraestrutura. O cliente precisa conseguir verificar que **a requisição veio de nós** e que **o payload não foi adulterado** no caminho. Também já tivemos um cliente que vazou secret em log da própria aplicação ([09:22] Diego).

## Decisão

- Assinar o **corpo do request** com **HMAC-SHA256** e enviar a assinatura no header `X-Signature`.
- **Uma secret por endpoint de webhook**, nunca global. A configuração guarda URL, secret, `customer_id` e estado ativo.
- A secret é **gerada pela plataforma** e devolvida ao cliente na criação do webhook.
- **Rotação via API:** o cliente pode pedir uma nova secret. A antiga continua válida em paralelo por **24 horas** e depois deixa de valer.
- A URL do webhook precisa ser **`https`**. Cadastro com `http` é recusado por validação no schema Zod.
- Sofia revisa o código de geração de secret e de HMAC antes do deploy (reservar dois dias úteis).

## Alternativas consideradas

### Secret global da plataforma
**Descartada:** se uma vazar, vazam todas. Com secret por endpoint, o vazamento fica contido a um cadastro.

### Rotação sem período de convivência
Trocar a secret na hora, invalidando a antiga.
**Descartada implicitamente:** o cliente precisa de tempo para migrar os sistemas dele. Por isso foi definido o grace period de 24h.

## Consequências

**Positivas**
- Algoritmo padrão de mercado, com biblioteca disponível para qualquer cliente.
- Raio de impacto de um vazamento limitado a um endpoint.
- A rotação permite resposta rápida a vazamento sem quebrar a integração.

**Negativas / trade-offs**
- Durante as 24h de convivência, o envio precisa considerar duas secrets válidas (ver detalhes no FDD), o que dá mais complexidade de modelo e de código.
- A secret precisa ser armazenada de forma recuperável para assinar, e por isso não pode ser só um hash. Exige cuidado extra de acesso e de log (o logger já tem redaction, mas a secret precisa entrar nela).
- A verificação depende do cliente implementar a validação corretamente.

## Referências

- Transcrição: [09:19]–[09:23] (Sofia, Bruno, Diego), [09:31] (Marcos), [09:46] (Sofia)
- Código: `src/shared/logger/index.ts` (`redactPaths`), padrão de schemas Zod em `src/modules/*/*.schemas.ts`
