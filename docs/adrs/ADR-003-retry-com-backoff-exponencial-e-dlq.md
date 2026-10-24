# ADR-003: Retry com backoff exponencial (1m/5m/30m/2h/12h) e Dead Letter Queue em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisões em [09:17], [09:19] e [09:36])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Consultados:** Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar lento ou fora do ar ([09:14] Larissa). Já houve cliente com duas horas de indisponibilidade em manutenção planejada ([09:16] Diego). Precisamos insistir o suficiente para cobrir esses casos sem deixar evento pendurado para sempre ([09:15] Diego). Também precisamos de um caminho de investigação e reprocessamento para o que falhar de vez.

## Decisão

1. **Backoff exponencial com teto de tentativas:** depois do teto, a falha é considerada permanente e o evento vai para a DLQ ([09:15] Diego).
2. **Agenda de retentativas:** 1 min → 5 min → 30 min → 2 h → 12 h. São 5 tentativas na agenda, num total de quase 15 h entre a primeira falha e a última tentativa ([09:17] Diego, decidido em [09:17] Larissa).
   - Interpretação adotada: a entrega inicial **mais** 5 retentativas, uma para cada intervalo. Só assim fecham as "quase 15 horas" citadas. Ver a questão em aberto no [RFC](../RFC.md#5-questões-em-aberto).
3. **Timeout de 10 s:** o request HTTP tem timeout de 10 s. Um cliente que não responde nesse prazo conta como falha e entra no retry ([09:42] Diego).
4. **DLQ em tabela separada:** `webhook_dead_letter`, com payload, motivo da falha e timestamp, em vez de marcar `failed` na própria outbox ([09:18] Diego).
5. **Reprocessamento manual:** feito via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
   - O endpoint exige role `ADMIN` ([09:36] Sofia) e reaproveita o `requireRole` existente ([09:36] Larissa).
   - Quem fez o replay é registrado para auditoria ([09:36] Sofia).

## Alternativas Consideradas

### A1. Retry indefinido com backoff (descartada)

- **Prós:** nunca desiste de entregar.
- **Contras:** o evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego).

### A2. Apenas 3 tentativas (descartada)

- **Prós:** mais agressivo, falha mais rápido ([09:16] Bruno).
- **Contras:** três tentativas caberiam em cerca de 30 minutos e matariam o evento durante indisponibilidades comuns, como a manutenção de duas horas já observada ([09:16] Diego).

### A3. Marcar como `failed` na própria outbox, sem tabela de DLQ (descartada)

- **Prós:** uma tabela a menos.
- **Contras:** polui a leitura da outbox principal. A tabela separada é "mais limpa" e serve de evidência para debug e reprocessamento ([09:18] Diego).

## Consequências

### Positivas

- Cobre indisponibilidades de até ~15 h, aceitas pelo produto ([09:17] Marcos).
- Nenhum evento fica eternamente pendente: todo evento termina **entregue** ou **na DLQ**.
- A DLQ preserva payload e motivo, o que viabiliza diagnóstico e replay auditado.

### Negativas

- **Atraso longo:** um evento pode chegar com até ~15 h de atraso, depois que o cliente volta.
- **Replay manual:** depende de um ADMIN, porque não há reprocessamento automático da DLQ.
- **Sem aviso ao cliente:** o aviso proativo de falhas (ex.: e-mail) foi adiado para a próxima fase ([09:37] Larissa). O cliente só descobre pelo histórico de entregas.

### Trade-off explícito

Escolhemos **persistência razoável (~15 h)** em vez de **persistência infinita**, aceitando que falhas mais longas exigem intervenção humana via DLQ.
