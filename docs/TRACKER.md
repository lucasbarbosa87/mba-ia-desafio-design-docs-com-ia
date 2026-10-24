# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado nos documentos ([PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md) e [ADRs](adrs/README.md)) e a sua origem.

**Como ler:**

- **Fonte `TRANSCRICAO`:** a localização é `[hh:mm] Falante` em `TRANSCRICAO.md`. Quando um item foi construído ao longo de várias falas, a tabela aponta a fala **decisiva** (quem fechou ou formulou o ponto). As falas complementares aparecem no próprio documento.
- **Fonte `CODIGO`:** a localização é o caminho de um arquivo que existe no repositório.
- **Itens marcados "(proposta FDD)" nos documentos:** a fonte indicada é a decisão ou o trecho de código de que a proposta deriva.

## PRD (`docs/PRD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas, MaxDistribuição e Nova Cargo por notificação de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes fazem polling em `GET /orders`, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | Risco de a Atlas migrar para o concorrente se não sair até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Restrição | "Tempo real" significa < 10 s para os clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Restrição | Webhooks só outbound (nós → cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-06 | docs/PRD.md | Contexto | Ciclo de vida do pedido e transições permitidas | CODIGO | src/modules/orders/order.status.ts |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência commit → primeira tentativa < 10 s (p95) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | 3 de 3 clientes com webhook em produção até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | 100% das mudanças de status assinadas geram evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Janela de ~15 h coberta por retentativas | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Entrega em 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| PRD-SCN-01 | docs/PRD.md | Cenário de uso | Cadastro com filtro SHIPPED/DELIVERED e secret na criação | TRANSCRICAO | [09:33] Marcos |
| PRD-SCN-02 | docs/PRD.md | Cenário de uso | Cliente indisponível 2 h tem eventos retentados | TRANSCRICAO | [09:16] Diego |
| PRD-SCN-03 | docs/PRD.md | Cenário de uso | Rotação de secret após vazamento | TRANSCRICAO | [09:22] Diego |
| PRD-SCN-04 | docs/PRD.md | Cenário de uso | Consulta ao histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-SCN-05 | docs/PRD.md | Cenário de uso | ADMIN faz replay de evento da DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo (adiado) | E-mail ao cliente em caso de falhas repetidas | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo (descartado) | Dashboard visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo (adiado) | Rate limiting de envio | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo (descartado) | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo (adiado) | Múltiplos workers / ordem global | TRANSCRICAO | [09:13] Diego |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo (adiado) | Endurecimento de permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com url, status desejados e customerId | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01a | docs/PRD.md | Restrição | `customerId` vem no body ou no path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status, aplicado na geração | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | POST ao endpoint do cliente a cada mudança de status assinada | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Conteúdo do payload (ids, tipo, timestamp, status, total; sem items) | TRANSCRICAO | [09:43] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do corpo com secret do endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Webhook-Id, X-Timestamp | TRANSCRICAO | [09:44] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Rotação de secret com validade da antiga por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Retry automático com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | DLQ com payload, motivo e data | TRANSCRICAO | [09:18] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Replay de DLQ por ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Replay registra quem executou (auditoria) | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s; polling de 2 s aceito | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Consistência status ↔ evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não afeta mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | At-least-once com dedup por X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Ordem só por pedido e só com single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Só URLs https; http recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Secret por endpoint, nunca global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB com erro, sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Sem infra nova (MySQL existente) | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Fluxo de mudança de status do módulo de pedidos | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Usuários que representam o cliente usam JWT da plataforma | TRANSCRICAO | [09:32] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Agendamento da revisão de segurança | TRANSCRICAO | [09:49] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Cliente precisa deduplicar por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso leva Atlas ao concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente processa duplicata | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Indisponibilidade > 15 h leva eventos para a DLQ | TRANSCRICAO | [09:17] Marcos |
| PRD-RISK-05 | docs/PRD.md | Risco | Rajadas sem rate limit | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Qualquer role autenticada gerencia qualquer webhook | TRANSCRICAO | [09:37] Sofia |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | Evento entregue em < 10 s para status assinado | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceitação | Status não assinado não gera evento | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-04 | docs/PRD.md | Critério de Aceitação | Secret só na criação e na rotação | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-05 | docs/PRD.md | Critério de Aceitação | Cliente valida X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de Aceitação | Secret antiga válida por 24h após rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de Aceitação | Retry na agenda e DLQ com motivo | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-08 | docs/PRD.md | Critério de Aceitação | Replay só ADMIN, com registro de autor | TRANSCRICAO | [09:36] Larissa |
| PRD-AC-09 | docs/PRD.md | Critério de Aceitação | Falha na gravação do evento impede a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-10 | docs/PRD.md | Critério de Aceitação | Histórico mostra as últimas 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| PRD-TST-01 | docs/PRD.md | Estratégia de Teste | Testes de integração com vitest + supertest | CODIGO | tests/orders.test.ts |
| PRD-TST-02 | docs/PRD.md | Estratégia de Teste | Integração no order.service e testes ponta a ponta no plano | TRANSCRICAO | [09:46] Larissa |
| PRD-TST-03 | docs/PRD.md | Estratégia de Teste | Validação com clientes após comunicação | TRANSCRICAO | [09:49] Marcos |

## RFC (`docs/RFC.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-META-01 | docs/RFC.md | Metadado | Revisores = participantes da reunião | TRANSCRICAO | [09:00] Larissa |
| RFC-CTX-01 | docs/RFC.md | Restrição | Time pequeno, sem apetite para infra nova | TRANSCRICAO | [09:07] Diego |
| RFC-CTX-02 | docs/RFC.md | Restrição | Mudança de status não pode depender do cliente | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-03 | docs/RFC.md | Restrição | Cliente precisa provar a origem e a integridade do envio | TRANSCRICAO | [09:19] Sofia |
| RFC-CTX-04 | docs/RFC.md | Restrição | Prazo de 3 sprints com revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| RFC-CTX-05 | docs/RFC.md | Contexto | Transação única do `changeStatus` (orders, history, estoque) | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Proposta | Módulo webhooks seguindo a estrutura dos domínios | TRANSCRICAO | [09:27] Bruno |
| RFC-PROP-02 | docs/RFC.md | Proposta | Publicação transacional recebendo o client da transação | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-03 | docs/RFC.md | Proposta | Worker como novo entry point, instância única | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-04 | docs/RFC.md | Proposta | Contrato de entrega: headers, timeout e limite | TRANSCRICAO | [09:44] Diego |
| RFC-PROP-05 | docs/RFC.md | Proposta | https obrigatório e revisão de segurança | TRANSCRICAO | [09:23] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Envio HTTP síncrono no `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila dedicada | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido ou 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-OQ-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| RFC-OQ-02 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers (partição ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-OQ-03 | docs/RFC.md | Questão em aberto | E-mail de alerta na próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OQ-04 | docs/RFC.md | Questão em aberto | Endurecimento de permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-OQ-04a | docs/RFC.md | Observação | `User` sem vínculo com `Customer` no schema | CODIGO | prisma/schema.prisma |
| RFC-OQ-05 | docs/RFC.md | Questão em aberto | `customerId` no body ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-OQ-06 | docs/RFC.md | Questão em aberto | Arquivamento da outbox | TRANSCRICAO | [09:08] Diego |
| RFC-OQ-07 | docs/RFC.md | Questão em aberto | Interpretação de "5 tentativas" x "quase 15 horas" | TRANSCRICAO | [09:17] Diego |
| RFC-OQ-08 | docs/RFC.md | Questão em aberto | Formato da assinatura durante a rotação | TRANSCRICAO | [09:21] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Falha nos inserts da outbox aborta a mudança de status | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Clientes precisam validar HMAC e deduplicar; doc no portal | TRANSCRICAO | [09:26] Marcos |
| RFC-RISK-01 | docs/RFC.md | Risco | Crescimento da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Worker único como gargalo | TRANSCRICAO | [09:12] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Secret armazenada de forma recuperável | TRANSCRICAO | [09:46] Sofia |
| RFC-RISK-04 | docs/RFC.md | Risco | Duplicatas no cliente | TRANSCRICAO | [09:24] Diego |
| RFC-NEXT-01 | docs/RFC.md | Próximo passo | Sessão de revisão com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |

## ADRs (`docs/adrs/`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL na mesma transação do status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Transação do `changeStatus` em `prisma.$transaction` | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Envio síncrono descartado | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Streams descartado (overengineering) | TRANSCRICAO | [09:07] Larissa |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Índices em status e created_at; arquivamento fora de escopo | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-D3 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Entry point `src/worker.ts` (novo) e `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D4 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D4a | docs/adrs/ADR-002-worker-separado-em-polling.md | Referência de código | `createPrismaClient()` | CODIGO | src/config/database.ts |
| ADR-002-D5 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Ordem só por order_id e single-worker | TRANSCRICAO | [09:12] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa | Trigger de banco (MySQL sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa | Múltiplos workers adiado | TRANSCRICAO | [09:13] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D4 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela `webhook_dead_letter` | TRANSCRICAO | [09:18] Diego |
| ADR-003-D5 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | Só 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| ADR-003-CONS-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | ~15 h de indisponibilidade aceitáveis | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D1 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | Algoritmo SHA-256 (padrão de mercado) | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D2 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Decisão | Configuração armazena url, secret, customer_id e ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-ALT-01 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Alternativa | Secret global descartada | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONS-01 | docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Vazamento prévio de secret em log de cliente | TRANSCRICAO | [09:22] Diego |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-D2 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | UUID gerado na entrada na outbox, único por evento | TRANSCRICAO | [09:25] Diego |
| ADR-005-D3 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar fica com o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Módulo `src/modules/webhooks` (novo) no padrão dos domínios | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Prefixo WEBHOOK_ nos códigos de erro | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D4 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Pino e error middleware sem nada novo | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D6 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | UUID na outbox seguindo o projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | `AppError` com statusCode e errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | `NotFoundError` fixa NOT_FOUND; classes de domínio com código | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | `requireRole` existente | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Referência de código | ZodError convertido em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa | ID auto incremental na outbox | TRANSCRICAO | [09:51] Diego |
| ADR-007 | docs/adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-D2 | docs/adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md | Decisão | Payload enxuto sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-D3 | docs/adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md | Decisão | Filtro de eventos na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-007-D4 | docs/adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md | Restrição | Limite de 64KB como requisito não funcional | TRANSCRICAO | [09:24] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md | Alternativa | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT-03 | docs/adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md | Alternativa | Filtrar no envio | TRANSCRICAO | [09:34] Diego |

## FDD (`docs/FDD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Evento na mesma transação do `changeStatus` | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Entrega < 10 s com polling de 2 s | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | 100% dos envios assinados | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Todo evento termina entregue ou na DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Sem infra nova; PrismaClient próprio | TRANSCRICAO | [09:30] Bruno |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Erros no envelope padrão com WEBHOOK_* | TRANSCRICAO | [09:29] Larissa |
| FDD-ESC-01 | docs/FDD.md | Exclusão | Criação de pedido não emite evento (único gancho é `changeStatus`) | TRANSCRICAO | [09:40] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Tabela de configuração (url, secret, customer_id, ativo) | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Outbox com índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | eventId gerado na aplicação com pacote `uuid` | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | orderId sem FK (delete de pedido CANCELLED) | CODIGO | src/modules/orders/order.service.ts |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | Payload serializado como texto para HMAC estável | TRANSCRICAO | [09:22] Sofia |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | DLQ com payload, motivo, timestamp e auditoria de replay | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-07 | docs/FDD.md | Modelo de dados | Uma linha de outbox por webhook destino | TRANSCRICAO | [09:44] Sofia |
| FDD-DADOS-08 | docs/FDD.md | Modelo de dados | IDs UUID Char(36) como nos models existentes | CODIGO | prisma/schema.prisma |
| FDD-FLX-01 | docs/FDD.md | Fluxo | Busca webhooks ativos do customer dentro do tx | TRANSCRICAO | [09:41] Bruno |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Sem assinante, não insere | TRANSCRICAO | [09:34] Bruno |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-FLX-04 | docs/FDD.md | Fluxo | Inserção com status PENDING | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Exceção propaga e reverte a transação | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-06 | docs/FDD.md | Fluxo | Seleção dos pendentes mais antigos | TRANSCRICAO | [09:09] Diego |
| FDD-FLX-07 | docs/FDD.md | Fluxo | Ordem por (webhook, pedido) na seleção | TRANSCRICAO | [09:13] Larissa |
| FDD-FLX-08 | docs/FDD.md | Fluxo | Recuperação de PROCESSING no startup | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-09 | docs/FDD.md | Fluxo | Shutdown gracioso como em `src/server.ts` | CODIGO | src/server.ts |
| FDD-FLX-10 | docs/FDD.md | Fluxo | Webhook inativo vai para a DLQ | TRANSCRICAO | [09:21] Bruno |
| FDD-FLX-11 | docs/FDD.md | Fluxo | Sem secret não envia (WEBHOOK_SECRET_REQUIRED) | TRANSCRICAO | [09:28] Bruno |
| FDD-FLX-12 | docs/FDD.md | Fluxo | > 64KB não envia | TRANSCRICAO | [09:24] Larissa |
| FDD-FLX-13 | docs/FDD.md | Fluxo | Assinatura com secret atual e anterior na janela | TRANSCRICAO | [09:21] Sofia |
| FDD-FLX-14 | docs/FDD.md | Fluxo | fetch com timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-FLX-15 | docs/FDD.md | Fluxo | Registro de cada tentativa no histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-FLX-16 | docs/FDD.md | Fluxo | 2xx marca como entregue | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-17 | docs/FDD.md | Fluxo | Falha incrementa tentativas e aplica retry | TRANSCRICAO | [09:42] Diego |
| FDD-FLX-18 | docs/FDD.md | Fluxo | Cálculo de nextAttemptAt pela agenda | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-19 | docs/FDD.md | Fluxo | Agenda esgotada vai para a DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-FLX-20 | docs/FDD.md | Fluxo | Não-2xx, timeout e rede são retentáveis | TRANSCRICAO | [09:42] Diego |
| FDD-FLX-21 | docs/FDD.md | Fluxo | Mover para DLQ e limpar a outbox | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-22 | docs/FDD.md | Fluxo | Replay recoloca como pendente, com auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-FLX-23 | docs/FDD.md | Fluxo | Replay mantém o X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-FLX-24 | docs/FDD.md | Fluxo | Rotação com previousSecret por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLX-25 | docs/FDD.md | Fluxo | Duas assinaturas durante a janela | TRANSCRICAO | [09:21] Sofia |
| FDD-FLX-26 | docs/FDD.md | Fluxo | Uma secret anterior por vez | TRANSCRICAO | [09:21] Sofia |
| FDD-FLX-27 | docs/FDD.md | Fluxo | Secret gerada pela plataforma com node:crypto | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-01a | docs/FDD.md | Contrato | Prefixo `/api/v1` das rotas | CODIGO | src/app.ts |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId= | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries (últimas 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | POST outbound com payload e headers | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-08a | docs/FDD.md | Contrato | Formato `ORD-NNNNNN` do order_number | CODIGO | src/modules/orders/order.service.ts |
| FDD-CONTRATO-08b | docs/FDD.md | Contrato | Envelope de erro `{ error: { code, message, details } }` | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Verificação HMAC pelo cliente sobre o corpo | TRANSCRICAO | [09:20] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_INVALID_EVENT_FILTER (PENDING inalcançável) | CODIGO | src/modules/orders/order.status.ts |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_INACTIVE | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:36] Sofia |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-11 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_NETWORK_ERROR | TRANSCRICAO | [09:14] Larissa |
| FDD-ERR-13 | docs/FDD.md | Erro | WEBHOOK_MAX_ATTEMPTS_EXCEEDED | TRANSCRICAO | [09:15] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Teto e DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Fallback: replay manual; e-mail adiado | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | Isolamento da API | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-06 | docs/FDD.md | Resiliência | Atomicidade com rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-RES-07 | docs/FDD.md | Resiliência | Recuperação de crash aceita reenvio | TRANSCRICAO | [09:24] Diego |
| FDD-RES-08 | docs/FDD.md | Resiliência | Proteção de tamanho de 64KB | TRANSCRICAO | [09:24] Larissa |
| FDD-RES-09 | docs/FDD.md | Resiliência | Sem seguir redirect (preserva https) | TRANSCRICAO | [09:23] Sofia |
| FDD-RES-10 | docs/FDD.md | Resiliência | Single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs de start/stop do worker | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de entrega com sucesso e latência | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log de falha com próxima tentativa | TRANSCRICAO | [09:15] Diego |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Log de envio para a DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Log de auditoria do replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Log de rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-OBS-07 | docs/FDD.md | Observabilidade | Snapshot periódico de backlog | TRANSCRICAO | [09:07] Bruno |
| FDD-OBS-07a | docs/FDD.md | Observabilidade | Redact de secret e assinatura nos logs | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-07b | docs/FDD.md | Observabilidade | Logger Pino com `redact` e nomes snake_case existentes | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-08 | docs/FDD.md | Métrica | Latência de entrega p95 < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-09 | docs/FDD.md | Métrica | Backlog da outbox | TRANSCRICAO | [09:07] Bruno |
| FDD-OBS-10 | docs/FDD.md | Métrica | Idade do evento pendente mais antigo | TRANSCRICAO | [09:10] Larissa |
| FDD-OBS-11 | docs/FDD.md | Métrica | Taxa de sucesso por webhook (insumo para e-mail e rate limit) | TRANSCRICAO | [09:39] Diego |
| FDD-OBS-12 | docs/FDD.md | Métrica | Itens de DLQ aguardando replay | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-13 | docs/FDD.md | Tracing | Correlação por eventId/requestId (sem lib de tracing na stack) | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | `changeStatus` chama `publishWebhookEvent(tx, ...)` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-01a | docs/FDD.md | Integração | Função pura recebendo o tx, sem injetar repository | TRANSCRICAO | [09:41] Diego |
| FDD-INT-02 | docs/FDD.md | Integração | Filtro de eventos validado contra transições | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Classes Webhook*Error no padrão existente | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-03a | docs/FDD.md | Integração | Exportação das novas classes | CODIGO | src/shared/errors/index.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Error middleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Validação estrutural via middleware; https no service | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | `authenticate` e `requireRole('ADMIN')` | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06a | docs/FDD.md | Integração | Padrão de rota com requireRole | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Novos redactPaths no logger | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Worker usa `createPrismaClient()` | CODIGO | src/config/database.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Novas variáveis no `envSchema` | CODIGO | src/config/env.ts |
| FDD-INT-10 | docs/FDD.md | Integração | `src/server.ts` como modelo do worker | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-11 | docs/FDD.md | Integração | Registro de controllers em `buildControllers` | CODIGO | src/app.ts |
| FDD-INT-11a | docs/FDD.md | Integração | Montagem das rotas `/webhooks` e `/admin/webhooks` | CODIGO | src/routes/index.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Novos models e migration | CODIGO | prisma/schema.prisma |
| FDD-INT-13 | docs/FDD.md | Integração | Scripts `worker` e `worker:start` | CODIGO | package.json |
| FDD-INT-14 | docs/FDD.md | Integração | `paginated()` na listagem | CODIGO | src/shared/http/response.ts |
| FDD-INT-15 | docs/FDD.md | Integração | Factory `createTestWebhook()` | CODIGO | tests/helpers/factories.ts |
| FDD-INT-16 | docs/FDD.md | Integração | Estrutura de arquivos do módulo e `webhook.worker.ts` | TRANSCRICAO | [09:28] Bruno |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node ≥ 20 (fetch, AbortSignal.timeout, crypto) | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Compatibilidade | Migration aditiva retrocompatível | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-DEP-03 | docs/FDD.md | Dependência | `uuid` já presente | CODIGO | package.json |
| FDD-DEP-04 | docs/FDD.md | Compatibilidade | Contratos existentes inalterados; 500 se outbox falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-DEP-05 | docs/FDD.md | Compatibilidade | Ordem de deploy: migration, API, worker | TRANSCRICAO | [09:11] Diego |
| FDD-DEP-06 | docs/FDD.md | Dependência | Revisão de segurança de 2 dias | TRANSCRICAO | [09:46] Sofia |
| FDD-DEP-07 | docs/FDD.md | Plano | Divisão das 3 sprints | TRANSCRICAO | [09:46] Larissa |
| FDD-AC-01 | docs/FDD.md | Critério técnico | Rollback do status se o publisher falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-02 | docs/FDD.md | Critério técnico | Status não assinado não insere linha | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério técnico | Envio < 10 s com headers e assinatura verificável | TRANSCRICAO | [09:44] Diego |
| FDD-AC-04 | docs/FDD.md | Critério técnico | Agenda de retry e DLQ após a 6ª falha | TRANSCRICAO | [09:17] Diego |
| FDD-AC-05 | docs/FDD.md | Critério técnico | Timeout de 10 s gera retry | TRANSCRICAO | [09:42] Diego |
| FDD-AC-06 | docs/FDD.md | Critério técnico | Payload > 64KB vai para a DLQ | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-07 | docs/FDD.md | Critério técnico | http e PENDING recusados com códigos WEBHOOK_ | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-08 | docs/FDD.md | Critério técnico | Secret fora de respostas e logs | TRANSCRICAO | [09:22] Diego |
| FDD-AC-09 | docs/FDD.md | Critério técnico | Duas assinaturas durante as 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-10 | docs/FDD.md | Critério técnico | Replay: 403 OPERATOR, 202 ADMIN, 409 duplicado | TRANSCRICAO | [09:36] Larissa |
| FDD-AC-11 | docs/FDD.md | Critério técnico | Ordem por pedido preservada com retry | TRANSCRICAO | [09:12] Diego |
| FDD-AC-12 | docs/FDD.md | Critério técnico | Reenvio após crash do worker | TRANSCRICAO | [09:24] Diego |
| FDD-AC-13 | docs/FDD.md | Critério técnico | Deliveries limitado a 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-RISK-01 | docs/FDD.md | Risco | Transação do `changeStatus` mais lenta | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Degradação da seleção com outbox grande | TRANSCRICAO | [09:07] Bruno |
| FDD-RISK-03 | docs/FDD.md | Risco | Secret recuperável no banco | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-04 | docs/FDD.md | Risco | X-Timestamp fora da assinatura | TRANSCRICAO | [09:44] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Head-of-line blocking por pedido | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-06 | docs/FDD.md | Risco | Worker único parado | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | CRUD aberto a qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| FDD-RISK-08 | docs/FDD.md | Risco | Sem endpoint de listagem de DLQ | TRANSCRICAO | [09:35] Diego |
| FDD-IMP-01 | docs/FDD.md | Proposta de implementação | Bloqueio de ordem por (webhook, pedido) | TRANSCRICAO | [09:13] Larissa |
| FDD-IMP-02 | docs/FDD.md | Proposta de implementação | Uma linha de outbox por webhook destino | TRANSCRICAO | [09:44] Sofia |
| FDD-IMP-03 | docs/FDD.md | Proposta de implementação | Payload como texto serializado | TRANSCRICAO | [09:52] Larissa |
| FDD-IMP-04 | docs/FDD.md | Proposta de implementação | orderId sem FK | CODIGO | src/modules/orders/order.service.ts |
| FDD-IMP-05 | docs/FDD.md | Proposta de implementação | Duas assinaturas na janela de rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-IMP-06 | docs/FDD.md | Proposta de implementação | customerId no body e em query | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-IMP-07 | docs/FDD.md | Proposta de implementação | Webhook inativo vai para a DLQ | TRANSCRICAO | [09:21] Bruno |
| FDD-IMP-08 | docs/FDD.md | Proposta de implementação | redirect manual | TRANSCRICAO | [09:23] Sofia |
| FDD-IMP-09 | docs/FDD.md | Proposta de implementação | 1 envio + 5 retentativas | TRANSCRICAO | [09:17] Diego |
