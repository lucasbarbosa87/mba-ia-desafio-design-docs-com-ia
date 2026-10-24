# ADR-001: Padrão Outbox no MySQL existente para eventos de webhook

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão fechada em [09:08])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Consultados:** Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-enxuto-na-insercao.md)

## Contexto

Três clientes B2B pediram para ser notificados quando o status dos seus pedidos muda ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de uma `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e mexe em `stock_quantity` dos produtos ([09:04] Bruno).

Precisamos registrar um evento para cada mudança de status relevante sem:

1. deixar a transação de status refém da latência ou da disponibilidade do cliente ([09:04] Bruno);
2. criar inconsistência entre "o status mudou" e "o evento foi registrado" ([09:40] Bruno, [09:41] Diego).

O time é pequeno e não opera hoje nenhum broker ou fila ([09:07] Diego). A stack atual é Node.js + Prisma + MySQL (`prisma/schema.prisma`).

## Decisão

Adotar o **padrão Transactional Outbox no MySQL existente**:

- Criar a tabela `webhook_outbox`. Dentro da **mesma transação** do `changeStatus`, inserir uma linha por evento a ser entregue ([09:06] Diego).
- Se a inserção na outbox falhar, a transação inteira faz rollback: não existe caso de status mudar sem evento ([09:40] Bruno).
- A tabela tem índices em `status` (pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego).
- Um processo separado consome a tabela (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- O arquivamento de linhas entregues (ex.: após ~30 dias) **fica fora do escopo** desta feature ([09:08] Diego).

## Alternativas Consideradas

### A1. Disparo HTTP síncrono dentro do `changeStatus` (descartada)

- **Prós:** implementação mais simples, sem tabela nem worker.
- **Contras:** um cliente lento travaria mudanças de status de outros pedidos ([09:04] Bruno). Se o cliente estiver fora do ar, não há resposta aceitável: dar rollback na mudança de status "não dá" ([09:04] Bruno). Diego: "Síncrono está fora de questão" ([09:06]).

### A2. Redis Streams (ou fila/broker equivalente) (descartada)

- **Prós:** mecanismo nativo de streaming, mais reativo, preparado para escala.
- **Contras:** exige subir e operar mais infra ([09:07] Larissa). Para um time pequeno, "subir Redis Cluster pra isso é overengineering" ([09:07] Diego). Além disso, publicar no Redis fora da transação MySQL reabre o problema de consistência (dual write) que a outbox elimina.

## Consequências

### Positivas

- **Atomicidade:** o evento existe se e somente se a mudança de status foi commitada ([09:06] Diego).
- **Zero infra nova:** mesmo banco, mesmo Prisma, mesma operação.
- **Isolamento:** a latência do cliente não afeta a API de pedidos.
- **Evidência:** a tabela vira um registro auditável do que foi emitido.

### Negativas

- **Latência mínima de polling:** o evento não sai no commit, sai no próximo ciclo do worker (até 2 s, [09:10] Larissa).
- **Crescimento da tabela:** sem arquivamento nesta fase, a tabela cresce indefinidamente ([09:08] Diego).
- **Transação mais pesada:** o `changeStatus` ganha mais um INSERT, além de um SELECT nos webhooks ativos do customer para filtrar ([09:34] Bruno).

### Trade-off explícito

Aceitamos **latência de alguns segundos e crescimento de tabela** em troca de **consistência forte entre status e evento** e de **nenhuma infraestrutura nova**. Para o requisito de "abaixo de 10 segundos" ([09:02] Marcos), essa latência é suficiente.
