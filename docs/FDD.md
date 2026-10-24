# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Pronto para implementação (após a sessão de revisão com Bruno e Diego, [09:50] Larissa) |
| **Responsáveis** | Bruno (módulo e integração com pedidos), Diego (worker e outbox), Sofia (revisão de segurança) |
| **Documentos** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

> Este documento descreve **como construir**. O porquê de cada decisão está nos ADRs e não é repetido aqui. Itens marcados como **(proposta FDD)** são detalhes de implementação que a reunião não fechou explicitamente. Eles derivam das decisões tomadas e estão listados na [seção 14](#14-decisões-de-implementação-propostas-neste-fdd) para validação na sessão de revisão.
>
> Arquivos marcados como **(novo)** ainda não existem no repositório e serão criados. Todos os outros caminhos citados existem no código base.

---

## 1. Contexto e motivação técnica

- O OMS não tem eventos nem filas. A única forma de um cliente saber que o pedido mudou de status é consultar `GET /api/v1/orders`.
- A mudança de status está concentrada em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que roda dentro de `this.prisma.$transaction`. Essa transação valida a transição (`canTransition` em `src/modules/orders/order.status.ts`), debita ou repõe estoque, atualiza `orders` e grava `order_status_history`.
- A feature acopla a emissão de eventos a essa transação, via **Outbox** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)), e transfere o I/O HTTP para um **worker separado** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Assim a transação de status continua rápida e independente dos clientes ([09:04] Bruno).

## 2. Objetivos técnicos

| ID | Objetivo |
| --- | --- |
| FDD-OBJ-01 | Gravar o evento na outbox **na mesma transação** do `changeStatus`. Se a gravação falhar, a transação inteira faz rollback ([09:40] Bruno). |
| FDD-OBJ-02 | Entregar o evento ao cliente em < 10 s em condições normais: polling de 2 s mais envio ([09:09] Diego). |
| FDD-OBJ-03 | Assinar 100% dos envios com HMAC-SHA256 por endpoint ([09:22] Sofia). |
| FDD-OBJ-04 | Garantir que todo evento termine **entregue** ou **na DLQ**, sem limbo ([09:15] Diego). |
| FDD-OBJ-05 | Nenhuma dependência de infra nova; o worker usa Prisma/MySQL com um client próprio ([09:30] Bruno). |
| FDD-OBJ-06 | Erros do módulo no envelope padrão, com códigos `WEBHOOK_*` ([09:29] Larissa). |

## 3. Escopo e exclusões

**Incluso:**

- módulo `src/modules/webhooks/` (novo) com CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ;
- função `publishWebhookEvent` chamada pelo `changeStatus`;
- entry point `src/worker.ts` (novo) e script `npm run worker`;
- 4 tabelas novas e uma migration Prisma.

**Excluído** (ver o [PRD §5.2](PRD.md#52-fora-de-escopo-descartado-ou-adiado-na-reunião)):

- e-mail de alerta ([09:37] Larissa);
- dashboard ([09:40] Larissa);
- rate limiting de saída ([09:39] Larissa);
- webhooks inbound ([09:02] Marcos);
- arquivamento de linhas antigas ([09:08] Diego);
- múltiplos workers ([09:13] Diego);
- eventos de **criação** de pedido. O único gancho discutido é o `changeStatus` ([09:40] Bruno); `OrderService.create` não emite evento.

## 4. Modelo de dados

Alterações em `prisma/schema.prisma`. Todos os ids são UUID `Char(36)`, como no resto do schema ([09:51] Larissa).

```prisma
enum WebhookOutboxStatus {
  PENDING     // pendente
  PROCESSING  // processando
  FAILED      // falhou, aguardando próxima tentativa
  DELIVERED   // entregue
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // OrderStatus[]: status que o webhook quer ouvir
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer          @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @db.Char(36) // = event_id = X-Event-Id (gerado na aplicação)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)     // sem FK: ver nota abaixo
  eventType     String              @db.VarChar(64)  // "order.status_changed"
  payload       String              @db.Text         // JSON serializado uma única vez (bytes assinados)
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([status])
  @@index([createdAt])
  @@index([webhookId, orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  eventId        String   @db.Char(36)
  attempt        Int
  success        Boolean
  requestPayload String   @db.Text
  responseStatus Int?
  responseBody   String?  @db.Text
  durationMs     Int
  errorCode      String?  @db.VarChar(64)
  createdAt      DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@index([eventId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  eventId      String    @db.Char(36)
  webhookId    String    @db.Char(36)   // sem FK: a evidência sobrevive à remoção do webhook
  orderId      String    @db.Char(36)
  eventType    String    @db.VarChar(64)
  payload      String    @db.Text
  reason       String    @db.VarChar(500)
  errorCode    String    @db.VarChar(64)
  attempts     Int
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  @@index([eventId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

No model `Customer` existente, adicionar a back-relation `webhooks WebhookEndpoint[]`. Isso não cria coluna nova em `customers`.

| ID | Nota de modelagem |
| --- | --- |
| FDD-DADOS-01 | A configuração guarda `url`, `secret`, `customer_id` e o estado ativo ([09:21] Bruno). |
| FDD-DADOS-02 | A outbox tem índices em `status` e `created_at` ([09:08] Diego). O índice `(webhookId, orderId)` sustenta a regra de ordem por pedido (FDD-FLX-07) **(proposta FDD)**. |
| FDD-DADOS-03 | `WebhookOutbox.id` é gerado na aplicação com `v4` do pacote `uuid`, já usado em `src/middlewares/request-logger.middleware.ts`. O mesmo valor vai no payload (`event_id`) e no header `X-Event-Id`, por isso precisa existir antes do INSERT ([09:25] Diego). |
| FDD-DADOS-04 | `orderId` **sem FK** para `orders`. `OrderService.delete` permite apagar pedidos em `CANCELLED`, e uma FK com cascade apagaria o evento de cancelamento ainda pendente **(proposta FDD)**. |
| FDD-DADOS-05 | `payload` é guardado como **texto serializado**, não como coluna `JSON`. O MySQL normaliza a ordem das chaves em colunas JSON, e precisamos que retentativas assinem exatamente os mesmos bytes **(proposta FDD)**. |
| FDD-DADOS-06 | DLQ em tabela separada, com payload, motivo e timestamp ([09:18] Diego). `replayedAt` e `replayedById` registram a auditoria do replay ([09:36] Sofia). |
| FDD-DADOS-07 | Uma linha de outbox por **(evento × webhook destino)**. Assim retry, DLQ e histórico são independentes por endpoint, e o `X-Webhook-Id` identifica o cadastro ([09:44] Sofia) **(proposta FDD)**. |

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro do `changeStatus`)

```
PATCH /api/v1/orders/:id/status
  └─ OrderService.changeStatus → prisma.$transaction(async (tx) => {
       1. findUnique order (+items)               ← já existe
       2. valida transição (canTransition)         ← já existe
       3. debita/repõe estoque                     ← já existe
       4. tx.order.update(status)                  ← já existe
       5. tx.orderStatusHistory.create(...)        ← já existe
       6. await publishWebhookEvent(tx, order, from, to)   ← NOVO
       7. findUnique refreshed                     ← já existe
     })
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` (em `src/modules/webhooks/webhook.publisher.ts`, novo) é uma função pura que recebe o `tx` ([09:41] Bruno, [09:41] Diego):

| ID | Passo |
| --- | --- |
| FDD-FLX-01 | Busca `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`. |
| FDD-FLX-02 | Filtra os endpoints cujo `events` contém `toStatus`. **Se nenhum, retorna sem inserir nada** ([09:34] Bruno). |
| FDD-FLX-03 | Para cada endpoint: gera `eventId = uuidv4()` e renderiza o **snapshot** do payload (§6.2) com `timestamp = new Date().toISOString()` ([09:52] Larissa). |
| FDD-FLX-04 | `tx.webhookOutbox.createMany(...)` com `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`. |
| FDD-FLX-05 | Qualquer exceção **não é capturada**: ela propaga, a `$transaction` faz rollback e a mudança de status não acontece ([09:40] Bruno). |

### 5.2 Processamento pelo worker

`src/worker.ts` (novo) faz o bootstrap e chama `runWebhookWorker()` de `src/modules/webhooks/webhook.worker.ts` (novo) ([09:28] Bruno).

```
startup:
  prisma = createPrismaClient()                         // client próprio, mesma DATABASE_URL
  UPDATE webhook_outbox SET status='PENDING' WHERE status='PROCESSING'   // recuperação (FDD-FLX-08)
loop (enquanto não receber SIGTERM/SIGINT):
  batch = selecionar elegíveis (FDD-FLX-06/07) LIMIT WEBHOOK_BATCH_SIZE
  UPDATE ... SET status='PROCESSING' WHERE id IN batch
  for evento in batch (sequencial, em ordem de created_at):
      entregar(evento)                                  // §5.3
  if batch.length < WEBHOOK_BATCH_SIZE: sleep(WEBHOOK_POLL_INTERVAL_MS = 2000)
```

| ID | Regra |
| --- | --- |
| FDD-FLX-06 | **Elegível:** `status IN (PENDING, FAILED) AND nextAttemptAt <= NOW()`, ordenado por `createdAt ASC` ([09:09] Diego, [09:12] Diego). |
| FDD-FLX-07 | **Ordem por pedido:** um evento só é elegível se **não existir** evento mais antigo **do mesmo `webhookId` e `orderId`** ainda na outbox com status diferente de `DELIVERED`. Sem isso, uma retentativa de `PAID` sairia depois do `SHIPPED` seguinte, quebrando a ordem por `order_id` decidida em [09:13] Larissa **(proposta FDD)**. Quando o evento antigo vai para a DLQ, ele sai da outbox e libera os seguintes. |
| FDD-FLX-08 | **Recuperação:** no startup, linhas em `PROCESSING` voltam a `PENDING`. Isso é seguro porque o worker é único ([09:12] Diego). Pode causar reenvio, o que é aceito pela garantia at-least-once ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). |
| FDD-FLX-09 | **Shutdown:** em `SIGTERM`/`SIGINT`, termina o evento corrente, sai do loop e chama `prisma.$disconnect()`, como em `src/server.ts`. |

### 5.3 Entrega de um evento

| ID | Passo | Resultado em caso de problema |
| --- | --- | --- |
| FDD-FLX-10 | Carrega o `WebhookEndpoint`. Se `active = false` → | DLQ com `WEBHOOK_INACTIVE`, sem retry **(proposta FDD)** |
| FDD-FLX-11 | Se o endpoint não tem `secret` → | DLQ com `WEBHOOK_SECRET_REQUIRED`. **Nunca enviar sem assinatura** ([09:28] Bruno) |
| FDD-FLX-12 | `body = outbox.payload` (string). Se `Buffer.byteLength(body) > 65536` → | DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem truncar e sem retry ([09:24] Larissa) |
| FDD-FLX-13 | Calcula `X-Signature` (§6.3) com `secret` e, se `previousSecretExpiresAt > now`, também com `previousSecret` | — |
| FDD-FLX-14 | `fetch(url, { method: 'POST', body, headers, signal: AbortSignal.timeout(10_000), redirect: 'manual' })` ([09:42] Diego) | Timeout → `WEBHOOK_DELIVERY_TIMEOUT`; erro de rede → `WEBHOOK_DELIVERY_NETWORK_ERROR`; não-2xx → `WEBHOOK_DELIVERY_FAILED` |
| FDD-FLX-15 | Grava `WebhookDelivery` com tentativa, sucesso, status e corpo da resposta, `durationMs` e `errorCode` ([09:34] Marcos) | — |
| FDD-FLX-16 | **2xx:** `status = DELIVERED`, `attempts += 1` | — |
| FDD-FLX-17 | **Falha:** `attempts += 1`, depois aplica a política de retry (§5.4) | — |

> A verificação de tamanho acontece **no worker**, não na inserção. Se acontecesse dentro do `changeStatus`, um payload grande faria rollback da mudança de status, o que é pior do que não enviar.

### 5.4 Retry com backoff

Agenda decidida: `RETRY_SCHEDULE = [1m, 5m, 30m, 2h, 12h]` ([09:17] Larissa).

| ID | Regra |
| --- | --- |
| FDD-FLX-18 | Depois da falha da tentativa `n` (`n = attempts` já incrementado): se `n <= RETRY_SCHEDULE.length`, então `status = FAILED`, `nextAttemptAt = now + RETRY_SCHEDULE[n-1]` e `lastError = errorCode`. |
| FDD-FLX-19 | Se `n > RETRY_SCHEDULE.length`, ou seja, falhou a 6ª chamada (1 inicial + 5 retentativas), o evento vai para a DLQ (§5.5) com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED`. Linha do tempo máxima: 1m + 5m + 30m + 2h + 12h ≈ 14h36min entre a primeira falha e a última tentativa ("quase 15 horas", [09:17] Diego). A interpretação de "5 tentativas" está registrada em [RFC-OQ-07](RFC.md#5-questões-em-aberto). |
| FDD-FLX-20 | Todo não-2xx é retentável, assim como timeout e erro de rede. Os erros permanentes (FDD-FLX-10 a 12) vão direto para a DLQ. |

### 5.5 DLQ e replay

| ID | Passo |
| --- | --- |
| FDD-FLX-21 | **Mover para DLQ** em uma transação: `webhookDeadLetter.create({ eventId: outbox.id, webhookId, orderId, eventType, payload, reason, errorCode, attempts })` e `webhookOutbox.delete({ id })`. A outbox principal fica limpa ([09:18] Diego). |
| FDD-FLX-22 | **Replay** (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`, só ADMIN), em uma transação: carrega o item; valida que não foi reprocessado e que o webhook existe e está ativo; insere na outbox **com o mesmo `id = eventId`**, `status = PENDING`, `attempts = 0` e o mesmo `payload`; grava `replayedAt = now` e `replayedById = req.user.id` ([09:18] Diego, [09:36] Sofia). |
| FDD-FLX-23 | O replay mantém o `X-Event-Id` original, então o cliente consegue deduplicar caso tenha processado o evento antes ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). Se falhar de novo, gera **um novo** item de DLQ. |

### 5.6 Rotação de secret

| ID | Passo |
| --- | --- |
| FDD-FLX-24 | `POST /api/v1/webhooks/:id/rotate-secret`: `previousSecret = secret`, `previousSecretExpiresAt = now + 24h` e `secret = novo valor` ([09:21] Sofia). |
| FDD-FLX-25 | Enquanto `previousSecretExpiresAt > now`, o worker envia **duas** assinaturas em `X-Signature` (nova e antiga). Depois disso, só a nova. Esse formato precisa ser validado pela Sofia ([RFC-OQ-08](RFC.md#5-questões-em-aberto)) **(proposta FDD)**. |
| FDD-FLX-26 | Uma nova rotação dentro da janela descarta a `previousSecret` corrente: só há uma secret anterior por vez **(proposta FDD)**. |
| FDD-FLX-27 | A secret é gerada com `crypto.randomBytes(32).toString('hex')`, do módulo nativo `node:crypto`, sem dependência nova ([09:31] Marcos: "secret é gerada pela gente"). |

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1`, montadas por `src/routes/index.ts`, e exigem `Authorization: Bearer <JWT>` (`authenticate`). Os erros seguem o envelope do `errorMiddleware`: `{ "error": { "code", "message", "details?" } }`.

### 6.1 API de configuração

#### FDD-CONTRATO-01: `POST /api/v1/webhooks` (criar webhook)

Qualquer role autenticada ([09:37] Sofia). O `customerId` vai no body, não vem do JWT ([09:32] Larissa).

```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "6f1c2a8e-1b7d-4e0a-9a4f-0c2d7e5b9a11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30",
  "customerId": "6f1c2a8e-1b7d-4e0a-9a4f-0c2d7e5b9a11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "9f2c4e1a7b3d5f8e0a6c2b4d8f1e3a5c7b9d0f2e4a6c8b1d3f5e7a9c0b2d4f6e",
  "createdAt": "2026-11-03T13:02:11.000Z",
  "updatedAt": "2026-11-03T13:02:11.000Z"
}
```

| Status | Quando |
| --- | --- |
| 201 | Criado. A `secret` aparece **somente aqui e na rotação** ([09:31] Marcos) |
| 400 | `VALIDATION_ERROR` (estrutura), `WEBHOOK_INVALID_URL` (não-https), `WEBHOOK_INVALID_EVENT_FILTER` |
| 401 | Token ausente ou inválido |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` |

#### FDD-CONTRATO-02: `GET /api/v1/webhooks?customerId=<uuid>&page=1&pageSize=20` (listar webhooks de um customer)

Listagem por customer ([09:33] Bruno), paginada com `paginated()` de `src/shared/http/response.ts`.

```http
GET /api/v1/webhooks?customerId=6f1c2a8e-1b7d-4e0a-9a4f-0c2d7e5b9a11
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30",
      "customerId": "6f1c2a8e-1b7d-4e0a-9a4f-0c2d7e5b9a11",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "previousSecretExpiresAt": null,
      "createdAt": "2026-11-03T13:02:11.000Z",
      "updatedAt": "2026-11-03T13:02:11.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
| --- | --- |
| 200 | Lista (sem `secret`) |
| 400 | `VALIDATION_ERROR` (`customerId` ausente ou não-UUID) |
| 401 | Não autenticado |

#### FDD-CONTRATO-03: `PATCH /api/v1/webhooks/:id` (editar webhook)

([09:33] Bruno) Pelo menos um campo deve ser enviado.

```http
PATCH /api/v1/webhooks/b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"], "active": true }
```

```http
HTTP/1.1 200 OK

{
  "id": "b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30",
  "customerId": "6f1c2a8e-1b7d-4e0a-9a4f-0c2d7e5b9a11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "previousSecretExpiresAt": null,
  "createdAt": "2026-11-03T13:02:11.000Z",
  "updatedAt": "2026-11-04T09:15:40.000Z"
}
```

| Status | Quando |
| --- | --- |
| 200 | Atualizado. Só vale para eventos gerados **depois** da alteração ([ADR-007](adrs/ADR-007-payload-snapshot-enxuto-na-insercao.md)) |
| 400 | `VALIDATION_ERROR`, `WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_EVENT_FILTER` |
| 404 | `WEBHOOK_NOT_FOUND` |

#### FDD-CONTRATO-04: `DELETE /api/v1/webhooks/:id` (remover webhook)

([09:33] Bruno)

```http
DELETE /api/v1/webhooks/b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 204 No Content
```

| Status | Quando |
| --- | --- |
| 204 | Removido. Eventos pendentes e histórico desse webhook são apagados em cascata; itens de DLQ são preservados como evidência |
| 404 | `WEBHOOK_NOT_FOUND` |

#### FDD-CONTRATO-05: `POST /api/v1/webhooks/:id/rotate-secret` (rotacionar secret)

([09:21] Sofia) Corpo vazio.

```http
POST /api/v1/webhooks/b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30/rotate-secret
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 200 OK

{
  "id": "b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30",
  "secret": "3a7c9e1b5d2f4a6c8e0b2d4f6a8c1e3b5d7f9a0c2e4b6d8f1a3c5e7b9d0f2a4c",
  "previousSecretExpiresAt": "2026-11-05T09:20:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| 200 | Nova secret emitida. A anterior vale até `previousSecretExpiresAt` (+24h) |
| 404 | `WEBHOOK_NOT_FOUND` |
| 409 | `WEBHOOK_INACTIVE`, se o webhook estiver desativado **(proposta FDD)** |

#### FDD-CONTRATO-06: `GET /api/v1/webhooks/:id/deliveries` (histórico de entregas)

Retorna as últimas 100 tentativas, da mais recente para a mais antiga ([09:34] Marcos).

```http
GET /api/v1/webhooks/b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30/deliveries
Authorization: Bearer eyJhbGciOi...
```

```http
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "0d5e7f91-3c2b-4a8d-9e6f-1b2c3d4e5f60",
      "eventId": "e3b1c7d2-8a4f-4b6e-9c0d-5f2a1e7b3c94",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "requestPayload": "{\"event_id\":\"e3b1c7d2-8a4f-4b6e-9c0d-5f2a1e7b3c94\",\"event_type\":\"order.status_changed\", ...}",
      "createdAt": "2026-11-03T14:06:02.000Z"
    },
    {
      "id": "7a1b2c3d-4e5f-4a6b-8c7d-9e0f1a2b3c4d",
      "eventId": "e3b1c7d2-8a4f-4b6e-9c0d-5f2a1e7b3c94",
      "attempt": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "requestPayload": "{\"event_id\":\"e3b1c7d2-8a4f-4b6e-9c0d-5f2a1e7b3c94\", ...}",
      "createdAt": "2026-11-03T14:05:01.000Z"
    }
  ]
}
```

| Status | Quando |
| --- | --- |
| 200 | Até 100 registros |
| 404 | `WEBHOOK_NOT_FOUND` |

#### FDD-CONTRATO-07: `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (replay de DLQ)

Exige `requireRole('ADMIN')` ([09:35] Diego, [09:36] Larissa). Corpo vazio.

```http
POST /api/v1/admin/webhooks/dead-letter/4c2e8a1f-6b3d-4f7a-9e0c-1d5b7a3f9e28/replay
Authorization: Bearer <JWT de ADMIN>
```

```http
HTTP/1.1 202 Accepted

{
  "deadLetterId": "4c2e8a1f-6b3d-4f7a-9e0c-1d5b7a3f9e28",
  "eventId": "e3b1c7d2-8a4f-4b6e-9c0d-5f2a1e7b3c94",
  "status": "PENDING",
  "replayedAt": "2026-11-04T10:00:00.000Z",
  "replayedById": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"
}
```

| Status | Quando |
| --- | --- |
| 202 | Recolocado na outbox; o envio acontece no próximo ciclo do worker |
| 401 / 403 | Não autenticado / `FORBIDDEN` para OPERATOR (erro existente do `requireRole`) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND`, `WEBHOOK_NOT_FOUND` |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`, `WEBHOOK_INACTIVE` |

### 6.2 Contrato de entrega ao cliente (outbound)

#### FDD-CONTRATO-08: `POST <url do cliente>`

```http
POST /oms HTTP/1.1
Host: hooks.atlascomercial.com.br
Content-Type: application/json
X-Event-Id: e3b1c7d2-8a4f-4b6e-9c0d-5f2a1e7b3c94
X-Webhook-Id: b8e0f3c4-5d2a-4f61-8c7e-2a9d1e4f6b30
X-Timestamp: 2026-11-03T14:05:01.312Z
X-Signature: sha256=5d41402abc4b2a76b9719d911017c592ae2d0f5b9e2c8a3e4f1a6b7c8d9e0f1a

{
  "event_id": "e3b1c7d2-8a4f-4b6e-9c0d-5f2a1e7b3c94",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-03T14:04:59.870Z",
  "order_id": "2d9f6b1a-7c3e-4a5b-8d0f-6e1a2b3c4d5e",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1c2a8e-1b7d-4e0a-9a4f-0c2d7e5b9a11",
  "total_cents": 18000
}
```

| Elemento | Semântica | Origem |
| --- | --- | --- |
| `event_id` / `X-Event-Id` | UUID gerado na inserção na outbox; estável em retries e replay; chave de deduplicação | [09:25] Diego |
| `event_type` | Sempre `order.status_changed` nesta fase | [09:43] Diego |
| `timestamp` (body) | Momento da mudança de status, ISO 8601 (snapshot) | [09:43] Diego, [09:52] Larissa |
| `order_number` | Formato `ORD-NNNNNN` gerado por `reserveOrderNumber` | `src/modules/orders/order.service.ts` |
| `total_cents` | Campo básico do pedido; **sem `items`** | [09:43] Diego |
| `X-Webhook-Id` | Id do cadastro que originou o envio | [09:44] Sofia |
| `X-Timestamp` | Momento do envio desta tentativa (ISO 8601), para o cliente detectar replay se quiser | [09:44] Diego |
| `X-Signature` | `sha256=<hex>` = HMAC-SHA256(secret, corpo bruto). Na janela de rotação: `sha256=<hex nova>,sha256=<hex antiga>` | [09:22] Sofia |
| Resposta 2xx | Sucesso, sem retry | [09:42] Diego |
| Qualquer outra resposta, timeout de 10 s ou erro de rede | Falha, entra em retry | [09:42] Diego |

### 6.3 Verificação pelo cliente (para o portal do desenvolvedor)

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

function isValid(rawBody: Buffer, header: string, secret: string): boolean {
  const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
  return header.split(',').some((part) => {
    const received = part.trim().replace(/^sha256=/, '');
    return received.length === expected.length &&
      timingSafeEqual(Buffer.from(received), Buffer.from(expected));
  });
}
```

A assinatura cobre **o corpo** do request ([09:22] Sofia). `X-Timestamp` **não** entra na assinatura, porque a decisão foi assinar o corpo. Ver o FDD-RISK-04.

## 7. Matriz de erros previstos

Todas as classes estendem `AppError` (`src/shared/errors/app-error.ts`) com prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa).

A reunião citou explicitamente só três códigos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, "etc."). Os demais seguem a regra do prefixo e derivam de comportamentos decididos (filtro de eventos, DLQ, replay, timeout, limite de 64KB). Os nomes são **(proposta FDD)** ([FDD-IMP-10](#14-decisões-de-implementação-propostas-neste-fdd)).

| ID | Código | HTTP | Classe | Onde ocorre | Comportamento |
| --- | --- | --- | --- | --- | --- |
| FDD-ERR-01 | `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` | PATCH, DELETE, rotate, deliveries, replay | Resposta de erro padrão |
| FDD-ERR-02 | `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError` | POST, PATCH | URL que não é `https` ([09:23] Sofia) |
| FDD-ERR-03 | `WEBHOOK_INVALID_EVENT_FILTER` | 400 | `WebhookInvalidEventFilterError` | POST, PATCH | `events` contém `PENDING`, que nenhuma transição produz em `order.status.ts` |
| FDD-ERR-04 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError` | POST, GET list | `customerId` inexistente |
| FDD-ERR-05 | `WEBHOOK_INACTIVE` | 409 / — | `WebhookInactiveError` | rotate, replay / worker | API: 409. Worker: evento vai para a DLQ sem retry |
| FDD-ERR-06 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError` | replay | — |
| FDD-ERR-07 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `WebhookDeadLetterAlreadyReplayedError` | replay | Impede replay duplo do mesmo item |
| FDD-ERR-08 | `WEBHOOK_SECRET_REQUIRED` | — | `WebhookSecretRequiredError` | worker | Endpoint sem secret: **não envia**, vai para a DLQ |
| FDD-ERR-09 | `WEBHOOK_PAYLOAD_TOO_LARGE` | — | `WebhookPayloadTooLargeError` | worker | > 64KB: não envia, vai para a DLQ sem retry ([09:24] Larissa) |
| FDD-ERR-10 | `WEBHOOK_DELIVERY_TIMEOUT` | — | (código de entrega) | worker | > 10 s, entra em retry ([09:42] Diego) |
| FDD-ERR-11 | `WEBHOOK_DELIVERY_FAILED` | — | (código de entrega) | worker | Resposta não-2xx, entra em retry; `responseStatus` é gravado |
| FDD-ERR-12 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | (código de entrega) | worker | DNS, conexão recusada ou TLS, entra em retry |
| FDD-ERR-13 | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | (código de entrega) | worker | Agenda esgotada, vai para a DLQ ([09:15] Diego) |

Erros que **não** levam prefixo, porque já são tratados por componentes existentes:

- `VALIDATION_ERROR` (estrutura do body, via `validate`);
- `UNAUTHORIZED` e `FORBIDDEN` (`authenticate` e `requireRole`);
- falha de INSERT na outbox: propaga como erro Prisma e cai no `500 INTERNAL_SERVER_ERROR` do `errorMiddleware`, revertendo a mudança de status.

Os códigos "de entrega" não são lançados como exceção HTTP. Eles ficam em `WebhookDelivery.errorCode`, `WebhookOutbox.lastError` e `WebhookDeadLetter.errorCode`.

## 8. Estratégias de resiliência

| ID | Estratégia | Valor | Origem |
| --- | --- | --- | --- |
| FDD-RES-01 | Timeout por chamada HTTP | 10 s (`AbortSignal.timeout`) | [09:42] Diego |
| FDD-RES-02 | Retries com backoff exponencial | 1m, 5m, 30m, 2h, 12h (5 retentativas) | [09:17] Larissa |
| FDD-RES-03 | Teto e destino final | Agenda esgotada → `webhook_dead_letter` | [09:15] Diego |
| FDD-RES-04 | Fallback operacional | Replay manual por ADMIN; o e-mail ao cliente foi **adiado** | [09:18] Diego, [09:37] Larissa |
| FDD-RES-05 | Isolamento da API | Envio fora da transação e fora do processo HTTP | [09:04] Bruno, [09:11] Diego |
| FDD-RES-06 | Atomicidade | Outbox na mesma `$transaction`; falha → rollback | [09:40] Bruno |
| FDD-RES-07 | Recuperação de crash | `PROCESSING` → `PENDING` no startup (reenvio aceito pelo at-least-once) | [09:24] Diego |
| FDD-RES-08 | Proteção de tamanho | 64KB, falha permanente | [09:24] Larissa |
| FDD-RES-09 | Sem redirect | `redirect: 'manual'`: um 3xx conta como falha, para não seguir redirecionamento para `http` **(proposta FDD)** | [09:23] Sofia |
| FDD-RES-10 | Single-worker | Um processo; escala horizontal fora de escopo | [09:12] Diego |

## 9. Observabilidade

O projeto já tem logger Pino estruturado e nenhuma biblioteca de métricas ou tracing (`package.json`). Nada novo será adicionado ([09:29] Bruno, [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)). Métricas e correlação saem de **logs estruturados** e de **consultas às tabelas**.

### 9.1 Logs (Pino, `src/shared/logger/index.ts`)

Os nomes de evento seguem o estilo existente, em snake_case como `server_started` e `http_request`.

| ID | Evento de log | Nível | Campos |
| --- | --- | --- | --- |
| FDD-OBS-01 | `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `batchSize`, `signal` |
| FDD-OBS-02 | `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `orderId`, `attempt`, `responseStatus`, `durationMs`, `latencyMs` (do `createdAt` da outbox até agora) |
| FDD-OBS-03 | `webhook_delivery_failed` | warn | os mesmos campos + `errorCode`, `nextAttemptAt` |
| FDD-OBS-04 | `webhook_dead_lettered` | error | `deadLetterId`, `eventId`, `webhookId`, `errorCode`, `attempts` |
| FDD-OBS-05 | `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, `userId` (auditoria, [09:36] Sofia) |
| FDD-OBS-06 | `webhook_secret_rotated` | info | `webhookId`, `userId`, `previousSecretExpiresAt` |
| FDD-OBS-07 | `webhook_worker_stats` (a cada ciclo com trabalho, ou a cada 30 ciclos ociosos) | info | `backlog`, `oldestPendingAgeMs`, `deadLetterCount` |

**Redação:** adicionar `'*.secret'`, `'*.previousSecret'` e `'headers["x-signature"]'` aos `redactPaths`. O motivo é o vazamento de secret em log já ocorrido com um cliente ([09:22] Diego).

### 9.2 Métricas (derivadas de logs e queries)

| ID | Métrica | Fonte | Uso / alerta sugerido |
| --- | --- | --- | --- |
| FDD-OBS-08 | `webhook_delivery_latency_ms` (p50/p95) | `latencyMs` em `webhook_delivery_succeeded` | Meta PRD-OBJ-01: p95 < 10 s |
| FDD-OBS-09 | `webhook_outbox_backlog` | `COUNT(*) WHERE status IN (PENDING, FAILED)` | Crescimento contínuo indica worker parado ou lento |
| FDD-OBS-10 | `webhook_outbox_oldest_pending_age_ms` | `NOW() - MIN(createdAt) WHERE status = PENDING` | > 10 s indica que o worker não acompanha |
| FDD-OBS-11 | `webhook_delivery_success_rate` | `webhook_deliveries.success` por webhook e período | Insumo para a decisão futura sobre e-mail e rate limit ([09:37] Larissa, [09:39] Larissa) |
| FDD-OBS-12 | `webhook_dead_letter_count` | `COUNT(*) WHERE replayedAt IS NULL` | Itens aguardando ação de ADMIN |

### 9.3 Tracing (correlação por ids)

Não há OpenTelemetry na stack, e adicioná-lo não foi discutido. A rastreabilidade ponta a ponta se faz por **ids de correlação** presentes em todos os logs:

```
http_request (requestId, PATCH /orders/:id/status)
   → order_status_history (orderId)
   → webhook_outbox (eventId, webhookId, orderId)
   → webhook_deliveries (eventId, attempt)
   → cliente recebe X-Event-Id / X-Webhook-Id
```

| ID | Item |
| --- | --- |
| FDD-OBS-13 | `eventId` é o trace id lógico do envio: aparece no log, na outbox, nas deliveries, na DLQ e no header recebido pelo cliente. É a chave para suporte ao cliente. |

## 10. Integração com o sistema existente

| ID | Arquivo (existente) | Como o módulo de webhooks se integra |
| --- | --- | --- |
| FDD-INT-01 | `src/modules/orders/order.service.ts` | Em `changeStatus`, depois de `tx.orderStatusHistory.create(...)` e antes do `findUnique` final, chamar `await publishWebhookEvent(tx, order, from, to)`. O `tx` é o `Prisma.TransactionClient` já tipado como `TxClient` no arquivo. Sem try/catch: a falha propaga e reverte a transação ([09:40] Bruno, [09:41] Bruno). O construtor de `OrderService` **não muda**, porque a função é importada diretamente e não há repository injetado ([09:41] Diego). |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | O mapa `transitions` define os status possíveis de `to`. Como nenhuma transição leva a `PENDING`, o schema de `events` rejeita `PENDING` (`WEBHOOK_INVALID_EVENT_FILTER`). Os valores válidos são `OrderStatus` exceto `PENDING`. |
| FDD-INT-03 | `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` | Adicionar as classes `Webhook*Error` seguindo o padrão de `InvalidStatusTransitionError` e `InsufficientStockError`, e exportar no `index.ts`. Como `NotFoundError` fixa o código `NOT_FOUND`, `WebhookNotFoundError` estende `AppError` (`src/shared/errors/app-error.ts`) diretamente: `super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND')`. |
| FDD-INT-04 | `src/middlewares/error.middleware.ts` | **Sem alteração.** Já serializa qualquer `AppError` como `{ error: { code, message, details } }` e trata `ZodError` e Prisma (`P2002`, `P2025`) ([09:29] Bruno). |
| FDD-INT-05 | `src/middlewares/validate.middleware.ts` | Usado nas rotas para a validação estrutural (`customerId` UUID, `url` string, `events` array não vazio de `OrderStatus`). Esse middleware converte qualquer `ZodError` em `VALIDATION_ERROR`. Por isso a regra **https** fica num schema Zod dedicado (`webhookUrlSchema` em `webhook.schemas.ts`, novo), executado pelo service com `safeParse`, que lança `WebhookInvalidUrlError` ([09:23] Sofia: "validação no schema Zod"; [09:28] Bruno: código `WEBHOOK_INVALID_URL`). |
| FDD-INT-06 | `src/middlewares/auth.middleware.ts` | Rotas `/webhooks` usam `authenticate`, como `buildOrderRouter`. A rota de replay usa `authenticate, requireRole('ADMIN')`, como `src/modules/users/user.routes.ts` ([09:36] Larissa). O `req.user.id` alimenta `replayedById` e os logs de auditoria. |
| FDD-INT-07 | `src/shared/logger/index.ts` | O worker e o módulo usam o `logger` exportado. Adicionar os paths de secret e assinatura em `redactPaths` (§9.1). |
| FDD-INT-08 | `src/config/database.ts` | O worker chama `createPrismaClient()` para obter **outra instância** no seu processo, com a mesma `DATABASE_URL` ([09:30] Bruno). A API continua usando o `prisma` exportado. |
| FDD-INT-09 | `src/config/env.ts` | Acrescentar ao `envSchema`, com defaults: `WEBHOOK_POLL_INTERVAL_MS` (2000), `WEBHOOK_HTTP_TIMEOUT_MS` (10000), `WEBHOOK_MAX_PAYLOAD_BYTES` (65536) e `WEBHOOK_BATCH_SIZE` (10, **proposta FDD**: "batch pequeno", [09:08] Diego). |
| FDD-INT-10 | `src/server.ts` | Serve de modelo para `src/worker.ts` (novo): bootstrap com `logger`, handlers de `SIGINT`/`SIGTERM`, `prisma.$disconnect()` e `process.exit` ([09:11] Larissa). |
| FDD-INT-11 | `src/app.ts` e `src/routes/index.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e `WebhookController` (novos). `Controllers` ganha `webhooks`. `buildApiRouter` monta `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. |
| FDD-INT-12 | `prisma/schema.prisma` | Novos models da §4, back-relation `webhooks` em `Customer` e migration via `npm run db:migrate`. |
| FDD-INT-13 | `package.json` | Novos scripts: `"worker": "tsx watch --env-file=.env src/worker.ts"` e `"worker:start": "node --env-file=.env dist/worker.js"`, no mesmo padrão de `dev` e `start` ([09:11] Larissa). Nenhuma dependência nova. |
| FDD-INT-14 | `src/shared/http/response.ts` | `paginated()` e `PaginatedResponse` na listagem de webhooks. |
| FDD-INT-15 | `tests/helpers/factories.ts` | Acrescentar `createTestWebhook()` ao lado de `createTestCustomer` e `createTestProduct`, para os testes de integração. |

**Estrutura de arquivos novos** ([09:27] Bruno, [09:28] Bruno):

```
src/worker.ts                                   (novo) entry point do worker
src/modules/webhooks/webhook.routes.ts          (novo) buildWebhookRouter + buildWebhookAdminRouter
src/modules/webhooks/webhook.controller.ts      (novo)
src/modules/webhooks/webhook.service.ts         (novo) CRUD, rotação, replay
src/modules/webhooks/webhook.repository.ts      (novo)
src/modules/webhooks/webhook.schemas.ts         (novo) Zod: create/update/list/params/webhookUrlSchema
src/modules/webhooks/webhook.publisher.ts       (novo) publishWebhookEvent(tx, order, from, to)
src/modules/webhooks/webhook.worker.ts          (novo) loop, seleção, entrega, retry, DLQ
src/modules/webhooks/webhook.signature.ts       (novo) HMAC-SHA256 e geração de secret
```

**Diff de referência do `changeStatus`:**

```ts
// src/modules/orders/order.service.ts
import { publishWebhookEvent } from '../webhooks/webhook.publisher.js';
// ...
      await tx.orderStatusHistory.create({ data: { orderId: id, fromStatus: from, toStatus: to, /* ... */ } });

      await publishWebhookEvent(tx, order, from, to); // NOVO: mesma transação, sem try/catch

      const refreshed = await tx.order.findUnique({ /* ... */ });
```

## 11. Dependências e compatibilidade

| ID | Item |
| --- | --- |
| FDD-DEP-01 | **Node ≥ 20** (`engines` no `package.json`): `fetch` global, `AbortSignal.timeout` e `node:crypto` nativos. Não adiciona cliente HTTP. |
| FDD-DEP-02 | **Prisma 5.22 / MySQL**: as tabelas novas entram por migration aditiva. Nenhuma coluna existente é alterada, então a migration é retrocompatível com a API atual. |
| FDD-DEP-03 | **`uuid` 11** (já presente): geração do `eventId` antes do INSERT. |
| FDD-DEP-04 | **Contratos existentes**: nenhum endpoint atual muda. `PATCH /orders/:id/status` ganha uma leitura e N inserts. O único modo de falha novo é um 500 se a outbox estiver indisponível, o que é intencional. |
| FDD-DEP-05 | **Ordem de deploy** **(proposta FDD)**: (1) migration, (2) API com o publisher, (3) worker. Eventos gerados antes do worker subir ficam `PENDING` e são enviados quando ele inicia, já que o worker é um processo independente ([09:11] Diego). |
| FDD-DEP-06 | **Revisão de segurança** da Sofia, com pelo menos 2 dias úteis antes do deploy ([09:46] Sofia). |
| FDD-DEP-07 | **Plano de entrega** ([09:46] Larissa): sprint 1, modelagem de outbox e DLQ; sprint 2, worker e retry; sprint 3, CRUD e deliveries (½), integração no `order.service` e testes E2E (½), HMAC, schemas e validações, mais a revisão da Sofia no fim. |

## 12. Critérios de aceite técnicos

| ID | Critério |
| --- | --- |
| FDD-AC-01 | Teste de integração: com o publisher forçado a lançar erro, `PATCH /orders/:id/status` retorna 500 e o pedido mantém o status anterior, sem linha nova em `order_status_history` nem em `webhook_outbox`. |
| FDD-AC-02 | Uma mudança para um status não assinado por nenhum webhook do customer não insere linha na outbox. |
| FDD-AC-03 | O worker envia em < 10 s depois do commit (servidor de teste local), com os 5 headers da §6.2 e `X-Signature` verificável pelo snippet da §6.3. |
| FDD-AC-04 | Com um servidor de teste respondendo 500, `nextAttemptAt` segue 1m/5m/30m/2h/12h. Depois da 6ª falha a linha sai da outbox e aparece em `webhook_dead_letter` com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED`. Usar relógio controlado nos testes. |
| FDD-AC-05 | Um servidor que demora 11 s gera `WEBHOOK_DELIVERY_TIMEOUT` e retry. |
| FDD-AC-06 | Um payload de 65 537 bytes não é enviado e vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry. |
| FDD-AC-07 | `POST /webhooks` com `http://` retorna 400 `WEBHOOK_INVALID_URL`; com `events: ["PENDING"]` retorna 400 `WEBHOOK_INVALID_EVENT_FILTER`. |
| FDD-AC-08 | A `secret` só aparece nas respostas de criação e rotação, e nunca nos logs (teste do `redact`). |
| FDD-AC-09 | Depois da rotação, `X-Signature` traz duas assinaturas até `previousSecretExpiresAt`, e uma só depois disso. |
| FDD-AC-10 | Replay como OPERATOR retorna 403 `FORBIDDEN`. Como ADMIN, retorna 202, a linha volta para a outbox com o **mesmo** `id` e `replayedById` fica preenchido. Um segundo replay do mesmo item retorna 409. |
| FDD-AC-11 | Dois eventos do mesmo pedido (`PAID` falhando, `PROCESSING` pendente) para o mesmo webhook: `PROCESSING` não é enviado antes de `PAID` ser entregue ou ir para a DLQ. |
| FDD-AC-12 | Um worker morto com linhas em `PROCESSING` as reenvia ao reiniciar. |
| FDD-AC-13 | `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, em ordem decrescente de `createdAt`. |

## 13. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| FDD-RISK-01 | A transação do `changeStatus` fica mais lenta (leitura de webhooks + inserts) | Baixa | Médio | Índice `(customerId, active)`; filtro em memória; `createMany` |
| FDD-RISK-02 | A query de elegibilidade com `NOT EXISTS` degrada com outbox grande e sem arquivamento | Média | Médio | Índices `status`, `createdAt` e `(webhookId, orderId)`; monitorar `oldest_pending_age`; arquivamento como próxima etapa ([09:08] Diego) |
| FDD-RISK-03 | Secret armazenada em claro no banco (necessária para assinar) | Média | Alto | `redact` no logger; secret nunca retornada fora de criação e rotação; criptografia em repouso a decidir na revisão da Sofia ([09:46] Sofia) |
| FDD-RISK-04 | `X-Timestamp` fora da assinatura: um atacante que capture um request pode reenviá-lo com outro timestamp | Baixa | Médio | TLS obrigatório reduz a captura; o cliente deduplica por `X-Event-Id`; incluir o timestamp no conteúdo assinado é tema para a revisão de segurança |
| FDD-RISK-05 | Head-of-line blocking: um evento com falha segura os seguintes do **mesmo pedido** por até ~15 h | Média | Baixo | Bloqueio limitado ao par (webhook, pedido); outros pedidos seguem; replay e DLQ liberam a fila |
| FDD-RISK-06 | Worker único parado = nenhuma entrega | Média | Alto | Alerta sobre `webhook_outbox_oldest_pending_age_ms`; supervisão do processo; recuperação de `PROCESSING` no startup |
| FDD-RISK-07 | Qualquer usuário autenticado altera webhooks de qualquer customer (`User` não tem vínculo com `Customer` em `prisma/schema.prisma`) | Média | Médio | Aceito nesta fase ([09:37] Sofia); replay restrito a ADMIN; logs com `userId` |
| FDD-RISK-08 | Não há endpoint de listagem da DLQ (não discutido); o ADMIN precisa do `deadLetterId` | Alta | Baixo | `deadLetterId` no log `webhook_dead_lettered`; avaliar endpoint de listagem na revisão |

## 14. Decisões de implementação propostas neste FDD

Itens que **não** foram fechados na reunião, derivados das decisões tomadas, a validar na sessão de revisão com Bruno e Diego ([09:50] Larissa):

| ID | Proposta | Derivada de |
| --- | --- | --- |
| FDD-IMP-01 | Bloqueio de ordem por (webhook, pedido) na seleção do worker (FDD-FLX-07) | Ordem por `order_id` ([09:13] Larissa) |
| FDD-IMP-02 | Uma linha de outbox por webhook destino (FDD-DADOS-07) | `X-Webhook-Id` ([09:44] Sofia) e histórico por webhook ([09:34] Marcos) |
| FDD-IMP-03 | Payload guardado como texto serializado (FDD-DADOS-05) | Snapshot ([09:52] Larissa) e HMAC sobre o corpo ([09:22] Sofia) |
| FDD-IMP-04 | `orderId` sem FK na outbox (FDD-DADOS-04) | `OrderService.delete` aceita `CANCELLED` (`src/modules/orders/order.service.ts`) |
| FDD-IMP-05 | Duas assinaturas em `X-Signature` na janela de rotação (FDD-FLX-25) | Grace period de 24h ([09:21] Sofia) |
| FDD-IMP-06 | `customerId` no body (POST) e em query (GET) | "body ou path" ([09:32] Larissa), filtro `customerId` em `listOrdersQuerySchema` (`src/modules/orders/order.schemas.ts`) |
| FDD-IMP-07 | Webhook inativo: evento vai para a DLQ, não é descartado (FDD-FLX-10) | Estado ativo na configuração ([09:21] Bruno) |
| FDD-IMP-08 | `redirect: 'manual'` (FDD-RES-09) | TLS obrigatório ([09:23] Sofia) |
| FDD-IMP-09 | Interpretação "1 envio + 5 retentativas" (FDD-FLX-19) | Agenda de 5 intervalos e "quase 15 horas" ([09:17] Diego) |
| FDD-IMP-10 | Nomes dos códigos `WEBHOOK_*` além dos três citados na reunião (§7) | Regra de prefixo ([09:29] Larissa) e exemplos ([09:28] Bruno) |
| FDD-IMP-11 | Rota `POST /webhooks/:id/rotate-secret` (o caminho não foi definido) | "Endpoint pro cliente conseguir pedir nova secret pela API" ([09:21] Sofia) |
