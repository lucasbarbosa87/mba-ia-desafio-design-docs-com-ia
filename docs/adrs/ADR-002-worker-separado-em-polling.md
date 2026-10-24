# ADR-002: Worker em processo separado, consumindo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão registrada em [09:10] e [09:11])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Consultados:** Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), falta decidir duas coisas:

- **como** o consumidor descobre que há eventos novos;
- **onde** esse consumidor roda.

O requisito de negócio é entregar em menos de 10 segundos ([09:02] Marcos). A aplicação hoje tem um único entry point HTTP, `src/server.ts`, que sobe o Express e trata `SIGINT`/`SIGTERM`.

## Decisão

1. **Polling em loop a cada 2 segundos:** buscar os eventos pendentes mais antigos em batch pequeno, processar e marcar ([09:09] Diego). A latência de 2 s no pior caso foi aceita ([09:10] Larissa).
2. **Processo separado da API:** o worker não roda dentro da instância HTTP, para que um restart da API não derrube o worker ([09:11] Diego).
3. **Novo entry point** `src/worker.ts` (arquivo novo), seguindo o modelo de `src/server.ts`, com script `npm run worker` ([09:11] Larissa). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` (arquivo novo, [09:28] Bruno).
4. **Mesmo banco, mesma stack, PrismaClient próprio:** mesma `DATABASE_URL`, mas uma instância nova de `PrismaClient`, porque o client é por processo ([09:11] Diego, [09:30] Bruno). A instância é criada com `createPrismaClient()` de `src/config/database.ts`.
5. **Single-worker:** a ordem de entrega é por `created_at` da outbox. Isso garante ordem **por `order_id`** enquanto houver um único worker. Ordem global **não** é garantida, e isso fica documentado como limitação conhecida ([09:12] Diego, [09:13] Larissa).

## Alternativas Consideradas

### A1. Trigger no banco para reagir à inserção (descartada)

- **Prós:** mais reativo, sem esperar o intervalo de polling ([09:09] Bruno).
- **Contras:** MySQL não tem listener nativo como o `NOTIFY/LISTEN` do Postgres. A trigger só executa SQL e não notifica processo externo. Avisar o worker exigiria improvisos, como escrever em arquivo ou chamar um endpoint ([09:09] Diego).

### A2. Worker dentro do processo da API (descartada)

- **Prós:** um único deploy e um único processo.
- **Contras:** se a API reinicia, o worker morre junto ([09:11] Diego), e cada réplica da API viraria um worker concorrente, o que quebra a premissa de single-worker.

### A3. Múltiplos workers em paralelo (adiada)

- **Prós:** mais vazão.
- **Contras:** perde a garantia de ordem por pedido. Exigiria particionar por `order_id` ou usar lock pessimista, o que foi tratado como "problema do futuro" ([09:13] Diego).

## Consequências

### Positivas

- O requisito de < 10 s é atendido com folga ([09:09] Diego).
- Deploy e ciclo de vida do worker ficam independentes da API.
- Nenhum componente novo de infraestrutura.

### Negativas

- **Custo de consulta constante:** uma query a cada 2 s mesmo sem eventos.
- **Vazão limitada:** limitada a um único processo.
- **Ordem limitada:** ordem apenas por `order_id` e só em single-worker. Escalar horizontalmente exigirá novo ADR.
- **Mais um processo:** mais um processo para monitorar e manter de pé em produção.

### Trade-off explícito

Trocamos **reatividade e escala horizontal** por **simplicidade e ordem previsível por pedido**. A latência de 2 s no pior caso é o preço aceito ([09:10] Larissa).
