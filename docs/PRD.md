# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks de notificação de mudança de status de pedidos (outbound) |
| **PM responsável** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião técnica, em detalhamento |
| **Prazo alvo** | Fim de novembro, estimado em 3 sprints ([09:45] Marcos, [09:47] Larissa) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

> Os IDs entre colchetes (ex.: `PRD-FR-01`) são as chaves usadas no [Tracker](TRACKER.md) para ligar cada item à sua origem.

---

## 1. Resumo e contexto da feature

O OMS controla o ciclo de vida dos pedidos (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED` possível até `PROCESSING`), mas **não tem nenhum mecanismo de notificação externa**. Hoje o cliente só descobre que um pedido mudou de status consultando a API.

Esta feature adiciona **webhooks outbound**. O cliente B2B cadastra uma URL e passa a receber um `POST` assinado sempre que um pedido dele muda para um status que ele assinou. O fluxo é só nosso → cliente. O cliente não envia webhooks para nós ([09:02] Marcos, [09:03] Sofia).

## 2. Problema e motivação

- **Pedido formal de três clientes B2B.** Atlas Comercial, MaxDistribuição e Nova Cargo pediram notificação em tempo real de mudança de status ([09:00] Marcos).
- **Integração lenta e cara.** Hoje esses clientes fazem polling periódico em `GET /orders`, o que deixa a integração lenta e cara para eles ([09:00] Marcos).
- **Risco comercial.** A Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos).
- **"Tempo real" para o cliente** significa menos de 10 segundos, sem precisar atualizar manualmente ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

**Público primário:** integradores técnicos de clientes B2B que consomem a API do OMS. O primeiro grupo é Atlas, MaxDistribuição e Nova Cargo.

**Público secundário:**

- **Administradores da plataforma (role `ADMIN`):** operam o reprocessamento de falhas ([09:36] Sofia).
- **Usuários operadores que representam o cliente:** cadastram webhooks pela API autenticada ([09:32] Marcos).

| # | Cenário |
| --- | --- |
| C1 | Um integrador cadastra a URL `https` do sistema dele, escolhe receber só `SHIPPED` e `DELIVERED` e guarda a secret devolvida na criação ([09:31] Marcos, [09:33] Marcos). |
| C2 | Um pedido do cliente passa para `SHIPPED`. Em poucos segundos o sistema do cliente recebe o evento assinado, valida o HMAC e atualiza o próprio painel sem polling. |
| C3 | O sistema do cliente fica 2 h fora do ar em manutenção. Os eventos são retentados automaticamente e entregues quando ele volta ([09:16] Diego). |
| C4 | A secret do cliente vazou em log. Ele pede rotação, recebe uma nova secret e tem 24h para migrar enquanto a antiga ainda vale ([09:21] Sofia, [09:22] Diego). |
| C5 | O integrador investiga por que não recebeu um evento e consulta o histórico das últimas entregas, com status, payload, resposta e tempo ([09:34] Marcos). |
| C6 | Um evento esgotou as tentativas e foi para a DLQ. Um ADMIN dispara o replay depois que o cliente corrigiu o endpoint ([09:18] Diego). |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| PRD-OBJ-01 | Notificação "em tempo real" na visão do cliente | Latência entre o commit da mudança de status e a primeira tentativa de entrega, com endpoint saudável | **< 10 s** (p95). O polling acrescenta no máximo ~2 s ([09:10] Larissa); o resto do orçamento é o tempo de resposta do cliente. Um endpoint que leva o timeout inteiro de 10 s é tratado como falha, não como "saudável" ([09:02] Marcos, [09:42] Diego) |
| PRD-OBJ-02 | Reter e atender os clientes que pediram a feature | Clientes do pedido original com webhook em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até **fim de novembro** ([09:00] Marcos, [09:45] Marcos) |
| PRD-OBJ-03 | Nenhuma mudança de status "silenciosa" | % de mudanças de status com webhook assinante que geraram evento na outbox | **100%**, garantido por transação atômica ([09:40] Bruno) |
| PRD-OBJ-04 | Tolerar indisponibilidade do cliente | Janela coberta por retentativas antes de ir para a DLQ | **≈ 15 h** (1m/5m/30m/2h/12h) ([09:17] Diego) |
| PRD-OBJ-05 | Entrega dentro do prazo negociado | Sprints até o deploy, incluindo revisão de segurança | **3 sprints** ([09:47] Larissa) |

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de webhooks por customer, via API autenticada.
- Filtro, por webhook, dos status de pedido que o cliente quer receber.
- Emissão de evento `order.status_changed` a cada mudança de status que algum webhook ativo do customer assine.
- Entrega assíncrona, assinada com HMAC-SHA256, com retry e Dead Letter Queue.
- Rotação de secret com período de convivência de 24h.
- Histórico de entregas por webhook.
- Replay manual de eventos da DLQ por ADMIN, com auditoria.

### 5.2 Fora de escopo (descartado ou adiado na reunião)

| ID | Item | Situação | Origem |
| --- | --- | --- | --- |
| PRD-OUT-01 | **Notificação por e-mail ao cliente** quando o webhook dele falha repetidamente | **Adiado** para a próxima fase, "depois que a gente medir o impacto" | [09:37] Larissa |
| PRD-OUT-02 | **Dashboard visual** para o cliente ver os webhooks | **Descartado** nesta fase. Só endpoints de API; o painel é projeto separado do time de frontend | [09:40] Larissa |
| PRD-OUT-03 | **Rate limiting de envio** por cliente | **Adiado** como "observar e decidir depois" | [09:39] Larissa |
| PRD-OUT-04 | **Webhooks inbound** (cliente enviando para nós) | **Descartado**, só outbound | [09:02] Marcos |
| PRD-OUT-05 | **Arquivamento** de eventos entregues (ex.: após 30 dias) | **Fora do escopo** desta feature | [09:08] Diego |
| PRD-OUT-06 | **Múltiplos workers em paralelo / ordem global** de eventos | **Adiado**, "problema do futuro" | [09:13] Diego |
| PRD-OUT-07 | **Endurecimento de permissões** do CRUD de webhooks (hoje qualquer role autenticada) | **Adiado**, "mais pra frente a gente pode endurecer" | [09:37] Sofia |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-FR-01 | O cliente cadastra um webhook informando `url`, a lista de status desejados e o `customerId`. O `customerId` **não** vem do JWT, vem no body ou no path ([09:32] Larissa). | [09:31] Marcos |
| PRD-FR-02 | A secret do webhook é **gerada pela plataforma** e devolvida **na criação**. | [09:31] Marcos |
| PRD-FR-03 | O cliente lista os webhooks de um customer. | [09:33] Bruno |
| PRD-FR-04 | O cliente edita um webhook (URL, filtro de eventos, estado ativo). | [09:33] Bruno |
| PRD-FR-05 | O cliente remove um webhook. | [09:33] Bruno |
| PRD-FR-06 | Cada webhook tem um **filtro de eventos** com a lista de status que quer ouvir (ex.: só `SHIPPED` e `DELIVERED`). O filtro é aplicado no momento em que o evento é gerado. | [09:33] Marcos, [09:34] Bruno |
| PRD-FR-07 | Quando um pedido muda de status, o sistema envia um `POST` JSON a cada webhook ativo do customer que assina o novo status. | [09:00] Marcos, [09:40] Bruno |
| PRD-FR-08 | O payload traz identificador do evento, tipo `order.status_changed`, timestamp ISO 8601, id e número do pedido, status de origem e destino, customer e campos básicos como o total. Não inclui itens. | [09:43] Diego |
| PRD-FR-09 | Cada envio leva a assinatura HMAC-SHA256 do corpo, com a secret do endpoint, para o cliente validar autenticidade e integridade. | [09:20] Sofia, [09:22] Sofia |
| PRD-FR-10 | Cada envio leva identificadores do evento e do webhook (`X-Event-Id`, `X-Webhook-Id`) e o timestamp do envio (`X-Timestamp`). | [09:44] Diego, [09:44] Sofia |
| PRD-FR-11 | O cliente pode **rotacionar a secret** pela API. A secret anterior continua válida por 24h. | [09:21] Sofia |
| PRD-FR-12 | Envios que falham são **retentados automaticamente** com backoff (1m, 5m, 30m, 2h, 12h). | [09:17] Larissa |
| PRD-FR-13 | Esgotadas as tentativas, o evento vai para uma **Dead Letter Queue** que guarda payload, motivo e data. | [09:18] Diego |
| PRD-FR-14 | Um **ADMIN** pode reprocessar (replay) um evento da DLQ, devolvendo-o à fila de envio. | [09:18] Diego, [09:36] Sofia |
| PRD-FR-15 | O replay registra **quem** o executou, para auditoria. | [09:36] Sofia |
| PRD-FR-16 | O cliente consulta o **histórico de entregas** de um webhook (últimas 100), com sucesso/falha, payload, resposta e tempo de resposta. | [09:34] Marcos |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-NFR-01 | **Latência:** evento entregue em menos de 10 s em condições normais. O polling de 2 s é aceito como latência mínima no pior caso. | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | **Consistência:** se o status mudou, o evento existe; se a mudança falhou, o evento não existe. | [09:06] Diego, [09:40] Bruno |
| PRD-NFR-03 | **Isolamento:** um cliente lento ou fora do ar não afeta a mudança de status de nenhum pedido. | [09:04] Bruno |
| PRD-NFR-04 | **Garantia at-least-once:** o cliente pode receber duplicatas e deduplica pelo `X-Event-Id`. | [09:24] Diego, [09:26] Larissa |
| PRD-NFR-05 | **Ordem:** garantida apenas por pedido e apenas enquanto houver um único worker. Não há ordem global. | [09:13] Larissa |
| PRD-NFR-06 | **Segurança de transporte:** só URLs `https`; `http` é recusado com erro de validação. | [09:23] Sofia |
| PRD-NFR-07 | **Segredo isolado:** cada endpoint tem secret própria, sem secret global. | [09:21] Sofia |
| PRD-NFR-08 | **Tamanho de payload:** limite de 64KB. Acima disso o envio não acontece e é tratado como erro, sem truncamento. | [09:24] Larissa |
| PRD-NFR-09 | **Timeout de envio:** 10 s por chamada. Sem resposta nesse prazo, conta como falha e entra em retry. | [09:42] Diego |
| PRD-NFR-10 | **Sem infra nova:** a solução usa o MySQL e a stack existentes, sem broker ou fila. | [09:07] Diego |
| PRD-NFR-11 | **Disponibilidade do processamento:** o worker roda em processo separado da API; restart da API não interrompe entregas. | [09:11] Diego |
| PRD-NFR-12 | **Revisão de segurança:** código de HMAC e geração de secret revisado pela Segurança, com pelo menos 2 dias úteis, antes do deploy. | [09:46] Sofia |

## 8. Decisões e trade-offs principais

Cada decisão está detalhada no ADR correspondente. Aqui fica só o impacto de produto.

| Decisão | Trade-off para o produto | ADR |
| --- | --- | --- |
| Outbox no MySQL em vez de envio síncrono ou Redis | Alguns segundos de latência em troca de consistência e de não subir infra nova | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado com polling de 2 s | Latência mínima de 2 s aceita pelo PM ([09:10] Marcos) | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15 h e depois DLQ | Clientes fora do ar por mais de ~15 h dependem de replay manual | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação de 24h | Mais gestão de secrets em troca de isolamento de vazamentos | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | O cliente precisa deduplicar; documentado no portal ([09:26] Marcos) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Consistência em vez de otimizações locais | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Payload enxuto como snapshot | Itens exigem consulta extra a `GET /orders/:id` | [ADR-007](adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md) |

## 9. Dependências

| ID | Dependência | Origem |
| --- | --- | --- |
| PRD-DEP-01 | O fluxo de mudança de status do módulo de pedidos é o ponto de geração dos eventos. | [09:40] Bruno |
| PRD-DEP-02 | Os usuários que representam o cliente se autenticam com o JWT da plataforma; não existe login próprio do cliente. | [09:32] Marcos |
| PRD-DEP-03 | Revisão de segurança da Sofia, agendada antes do deploy. | [09:49] Sofia |
| PRD-DEP-04 | Documentação no portal do desenvolvedor (integração e deduplicação), a cargo do PM. | [09:26] Marcos, [09:40] Marcos |
| PRD-DEP-05 | Os clientes precisam expor endpoint `https`, validar HMAC e deduplicar por `X-Event-Id`. | [09:23] Sofia, [09:25] Diego |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| PRD-RISK-01 | Atraso na entrega leva a Atlas a migrar para o concorrente ([09:00] Marcos) | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limit fora), estimativa de 3 sprints validada com o time e prazo confirmado com os clientes ([09:47] Marcos) |
| PRD-RISK-02 | Cliente processa o mesmo evento duas vezes por não deduplicar ([09:25] Sofia) | Alta | Médio | `X-Event-Id` estável entre tentativas e documentação em destaque no portal ([09:26] Marcos) |
| PRD-RISK-03 | Vazamento de secret pelo cliente, como já ocorreu antes em log ([09:22] Diego) | Média | Alto | Secret por endpoint e rotação via API com grace period de 24h ([09:21] Sofia) |
| PRD-RISK-04 | Cliente fora do ar por mais de ~15 h perde eventos para a DLQ | Baixa | Médio | DLQ persistente e replay por ADMIN ([09:18] Diego); histórico de entregas para o cliente diagnosticar |
| PRD-RISK-05 | Rajada de eventos sobrecarrega o endpoint do cliente, já que não há rate limit ([09:38] Diego) | Média | Médio | Monitorar volume por cliente e decidir rate limiting com dados ([09:39] Larissa) |
| PRD-RISK-06 | Qualquer usuário autenticado consegue gerenciar webhooks de qualquer customer ([09:37] Sofia) | Média | Médio | Aceito nesta fase; endurecimento de permissões previsto para depois. Replay da DLQ já restrito a ADMIN |

## 11. Critérios de aceitação

| ID | Critério |
| --- | --- |
| PRD-AC-01 | Dado um webhook ativo que assina `SHIPPED`, quando um pedido do customer muda para `SHIPPED`, o endpoint recebe um `POST` com o payload definido em menos de 10 s. |
| PRD-AC-02 | Dado um webhook que **não** assina `PAID`, quando um pedido muda para `PAID`, nenhum evento é gerado para esse webhook. |
| PRD-AC-03 | O cadastro com URL `http://` é recusado com erro de validação. |
| PRD-AC-04 | A secret aparece só na resposta de criação e na de rotação; listagem e consulta não a expõem. |
| PRD-AC-05 | O cliente consegue validar `X-Signature` recalculando o HMAC-SHA256 do corpo com a secret recebida. |
| PRD-AC-06 | Depois de uma rotação, envios feitos nas 24h seguintes são verificáveis tanto com a secret nova quanto com a antiga. Depois disso, só com a nova. |
| PRD-AC-07 | Com o endpoint do cliente fora do ar, o evento é retentado na agenda 1m/5m/30m/2h/12h e, esgotada, aparece na DLQ com o motivo. |
| PRD-AC-08 | Um ADMIN consegue fazer replay de um item da DLQ; um OPERATOR recebe `403`. O replay registra quem o fez. |
| PRD-AC-09 | Se a gravação do evento falhar, a mudança de status do pedido também não acontece. |
| PRD-AC-10 | O histórico de entregas mostra as últimas 100 tentativas com resultado, payload, resposta e tempo. |

## 12. Estratégia de testes e validação

- **Testes de integração da API** com `vitest` e `supertest`, já usados em `tests/orders.test.ts`, cobrindo o CRUD de webhooks, validações (`https`, filtro), autorização do replay (ADMIN x OPERATOR) e o histórico de entregas.
- **Teste de atomicidade:** simular falha na gravação do evento e verificar que o status do pedido não mudou (PRD-AC-09). Ponto crítico citado em [09:40] Bruno.
- **Testes do worker** contra um servidor HTTP de teste que responde `2xx`, erro, ou demora mais de 10 s, verificando retry, agenda de backoff, DLQ e limite de 64KB.
- **Teste ponta a ponta:** mudança de status → outbox → worker → endpoint de teste, medindo a latência contra a meta de < 10 s. A integração com o service de pedidos e os testes ponta a ponta estão previstos no plano de 3 sprints ([09:46] Larissa).
- **Validação de segurança:** revisão manual da Sofia sobre HMAC e geração de secret, com pelo menos 2 dias úteis antes do deploy ([09:46] Sofia).
- **Validação com clientes:** confirmar com Atlas, MaxDistribuição e Nova Cargo, após a publicação no portal, que conseguem verificar a assinatura e deduplicar ([09:49] Marcos).
