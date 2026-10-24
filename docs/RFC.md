# RFC: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Lucas Barbosa, a partir da reunião técnica conduzida por Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-10-23 |
| **Revisores** | Larissa (Tech Lead) · Marcos (PM) · Bruno (Eng. Pleno, Pedidos) · Diego (Eng. Sênior, Plataforma) · Sofia (Eng. de Segurança) |
| **Documentos** | [PRD](PRD.md) (por quê / o quê) · [FDD](FDD.md) (como construir) · [ADRs](adrs/README.md) (decisões) · [Tracker](TRACKER.md) |

---

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B sobre mudanças de status de pedidos por **webhooks outbound**. A arquitetura tem quatro peças:

- **Outbox:** o evento é gravado numa tabela **outbox no MySQL**, na mesma transação que muda o status. Se o status mudou, o evento existe.
- **Worker:** um **worker em processo separado** lê a outbox **a cada 2 s** e faz o `POST` assinado com **HMAC-SHA256** (secret por endpoint, rotacionável com 24h de convivência).
- **Falhas:** quando o envio falha, aplicamos **backoff exponencial** (1m/5m/30m/2h/12h). Esgotadas as tentativas, o evento vai para uma **DLQ** com replay manual por ADMIN.
- **Entrega:** a garantia é **at-least-once**, com deduplicação pelo cliente via `X-Event-Id`.

Tudo usa a stack e os padrões que já existem. **Nenhuma infraestrutura nova.**

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) fazem polling em `GET /orders` para descobrir mudanças de status, o que é lento e caro para eles. Eles pediram notificação "em tempo real", entendida como menos de 10 s ([09:00] Marcos, [09:02] Marcos). A Atlas ameaça migrar para o concorrente se a feature não sair até o fim do trimestre.

O OMS não tem mecanismo de eventos, filas ou notificação. A mudança de status acontece numa única transação no `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que já atualiza `orders`, grava `order_status_history` e mexe no estoque. Por isso ela é sensível a qualquer trabalho extra, principalmente I/O externo ([09:04] Bruno).

**Restrições que moldam a proposta:**

| ID | Restrição | Origem |
| --- | --- | --- |
| RFC-CTX-01 | Time pequeno, sem apetite para operar infra nova | [09:07] Diego |
| RFC-CTX-02 | A mudança de status não pode depender da disponibilidade do cliente | [09:04] Bruno |
| RFC-CTX-03 | O cliente precisa provar a origem e a integridade do envio | [09:19] Sofia |
| RFC-CTX-04 | Prazo de 3 sprints, incluindo revisão de segurança | [09:47] Larissa |

## 3. Proposta técnica

```
 API (processo HTTP)                          Worker (processo separado)
 ┌───────────────────────────────┐            ┌─────────────────────────────────┐
 │ PATCH /orders/:id/status      │            │ loop a cada 2s                  │
 │  └─ changeStatus ($transaction)│            │  ├─ lê eventos pendentes (batch)│
 │      ├─ update orders         │  MySQL     │  ├─ assina HMAC-SHA256          │
 │      ├─ insert status_history │──────────▶ │  ├─ POST https://cliente (10s)  │──▶ Cliente B2B
 │      ├─ estoque               │ webhook_   │  ├─ sucesso → entregue          │
 │      └─ insert webhook_outbox │ outbox     │  ├─ falha → agenda retry        │
 └───────────────────────────────┘            │  └─ esgotou → webhook_dead_letter│
                                              └─────────────────────────────────┘
```

**Componentes propostos:**

1. **Módulo `webhooks`:** segue a estrutura dos outros domínios. Expõe a API de configuração (CRUD, filtro de status, rotação de secret, histórico de entregas) e o replay de DLQ restrito a ADMIN ([09:27] Bruno, [09:36] Larissa).
2. **Publicação transacional:** o `changeStatus` chama uma função de publicação que recebe o client da transação corrente. Ela renderiza um **snapshot enxuto** do pedido e grava uma linha na outbox para cada webhook do customer que assina o novo status ([09:41] Bruno, [09:52] Larissa).
3. **Worker:** novo entry point, ao lado do servidor HTTP. Usa o mesmo banco, com PrismaClient próprio, e roda como **instância única**, o que mantém a ordem de entrega por pedido ([09:11] Larissa, [09:12] Diego).
4. **Contrato de entrega:**
   - `POST` JSON com evento `order.status_changed`;
   - headers de identificação (`X-Event-Id`, `X-Webhook-Id`), de autenticidade (`X-Signature`) e de envio (`X-Timestamp`);
   - timeout de 10 s e limite de 64KB ([09:42] Diego, [09:44] Diego, [09:24] Larissa).
5. **Segurança:** `https` obrigatório, secret por endpoint gerada pela plataforma e revisão de segurança antes do deploy ([09:23] Sofia, [09:46] Sofia).

Tabelas, contratos HTTP, algoritmos de seleção e retry, matriz de erros e pontos de integração com o código estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| ID | Alternativa | Trade-off que levou ao descarte | Origem |
| --- | --- | --- | --- |
| RFC-ALT-01 | **Envio HTTP síncrono dentro do `changeStatus`** | Um cliente lento travaria a mudança de status de outros pedidos, e um cliente fora do ar obrigaria a escolher entre rollback do status (inaceitável) ou perda do evento | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | **Redis Streams / fila dedicada** | Mais reativo, mas exige subir e operar infra nova ("overengineering" para um time pequeno) e reintroduz o problema de escrita dupla fora da transação | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-03 | **Trigger de banco para acordar o worker** | MySQL não tem `NOTIFY/LISTEN`; a trigger não notifica processo externo e exigiria gambiarras. O polling de 2 s já atende os 10 s | [09:09] Bruno, [09:09] Diego |
| RFC-ALT-04 | **Retry indefinido** ou **só 3 tentativas** | Indefinido deixa evento pendurado para sempre; 3 tentativas (~30 min) não cobrem manutenções de 2 h já observadas | [09:15] Diego, [09:16] Diego |
| RFC-ALT-05 | **Secret global da plataforma** | "Se vaza uma, vaza tudo" | [09:21] Sofia |
| RFC-ALT-06 | **Exactly-once** | Exige coordenação dos dois lados e é muito mais complexo; at-least-once com id de evento é o padrão de mercado | [09:25] Diego |

## 5. Questões em aberto

| ID | Questão | Situação | Origem |
| --- | --- | --- | --- |
| RFC-OQ-01 | **Rate limiting de saída por cliente.** Uma rajada de 50 mudanças de status por minuto vira 50 chamadas ao cliente? | Fora do escopo; "observar e decidir depois" | [09:38] Diego, [09:39] Larissa |
| RFC-OQ-02 | **Estratégia para escalar além de um worker.** As opções são particionar por `order_id` ou usar lock pessimista | Adiado ("problema do futuro"); limitação documentada | [09:13] Diego |
| RFC-OQ-03 | **Notificação por e-mail** quando o webhook do cliente falha repetidamente | Adiado para a próxima fase, depois de medir impacto | [09:37] Larissa |
| RFC-OQ-04 | **Endurecimento de permissões do CRUD.** Hoje qualquer role autenticada gerencia webhooks de qualquer customer. Observação: no `prisma/schema.prisma` atual, `User` não tem vínculo com `Customer` | Adiado ("mais pra frente") | [09:37] Sofia |
| RFC-OQ-05 | **`customerId` no body ou no path?** A reunião definiu só que não vem do JWT. O FDD propõe body na criação e query na listagem, a confirmar | Não decidido | [09:32] Larissa |
| RFC-OQ-06 | **Arquivamento da outbox** (~30 dias) | Fora do escopo desta feature | [09:08] Diego |
| RFC-OQ-07 | **Contagem de tentativas.** "5 tentativas" com 5 intervalos (1m…12h) e "quase 15 h" só fecha se forem 1 envio inicial + 5 retentativas. Confirmar essa interpretação | A confirmar com Diego/Larissa | [09:17] Diego, [09:17] Larissa |
| RFC-OQ-08 | **Rotação de secret.** Como um envio permanece "válido com a secret antiga" por 24h? O FDD propõe enviar as duas assinaturas no `X-Signature` durante a janela, para validação da Sofia | A validar na revisão de segurança | [09:21] Sofia, [09:46] Sofia |

## 6. Impacto e riscos

**Impacto no sistema existente**

- **Pedidos:** a transação do `changeStatus` ganha uma leitura (webhooks do customer) e N inserts na outbox. A falha desses inserts aborta a mudança de status, o que é intencional ([09:40] Bruno).
- **Banco:** quatro tabelas novas (configuração, outbox, histórico de entregas e DLQ) e uma migration Prisma.
- **Operação:** passa a existir um segundo processo (worker) para deploy e monitoramento.
- **Clientes:** precisam validar HMAC e deduplicar eventos, o que será documentado no portal ([09:26] Marcos).

**Riscos arquiteturais**

| ID | Risco | Mitigação |
| --- | --- | --- |
| RFC-RISK-01 | Crescimento contínuo da outbox e do histórico, sem arquivamento nesta fase | Índices em `status` e `created_at` ([09:08] Diego); arquivamento como próxima etapa (RFC-OQ-06) |
| RFC-RISK-02 | Worker único vira gargalo e ponto único de atraso | Polling de 2 s com folga sobre os 10 s; monitorar backlog; ADR futuro para multi-worker (RFC-OQ-02) |
| RFC-RISK-03 | Secret guardada de forma recuperável (necessária para assinar) | Secret por endpoint, `redact` no logger e revisão de segurança antes do deploy ([09:46] Sofia) |
| RFC-RISK-04 | Duplicatas no cliente (at-least-once) | `X-Event-Id` estável entre retentativas e replays; documentação no portal |

## 7. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s, single-worker |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada |
| [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com grace period de 24h |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | At-least-once com deduplicação via `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes (módulos, `AppError`, Pino, error middleware, `requireRole`, UUID) |
| [ADR-007](adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md) | Payload enxuto como snapshot na inserção, com filtro na inserção |

**Próximo passo:** sessão de revisão desta proposta com Bruno e Diego antes de começar a codar ([09:50] Larissa), e agendamento da revisão de segurança com a Sofia ([09:49] Sofia).
