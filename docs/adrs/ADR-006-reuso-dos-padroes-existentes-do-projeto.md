# ADR-006: Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão fechada em [09:30])
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos)
- **Consultados:** Diego (Plataforma), Sofia (Segurança)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

A codebase do OMS já tem convenções consolidadas:

| Padrão | Onde está no código |
| --- | --- |
| Módulo por domínio com `controller`, `service`, `repository`, `routes` e `schemas` | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/` etc. |
| Erros de domínio com código estável | `src/shared/errors/app-error.ts` (`AppError` com `statusCode` e `errorCode`) e `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`) |
| Tratamento centralizado de erros (AppError, ZodError, Prisma) | `src/middlewares/error.middleware.ts` |
| Validação com Zod via middleware | `src/middlewares/validate.middleware.ts` e `*.schemas.ts` |
| Autorização por role | `requireRole` em `src/middlewares/auth.middleware.ts` |
| Logger estruturado Pino com `redact` | `src/shared/logger/index.ts` |
| PrismaClient por processo | `createPrismaClient()` em `src/config/database.ts` |
| IDs UUID | `@id @default(uuid()) @db.Char(36)` em todos os models de `prisma/schema.prisma` |

A feature de webhooks pode seguir essas convenções ou introduzir estruturas próprias (bibliotecas novas, estrutura de pastas diferente, formato de erro próprio).

## Decisão

Reusar ao máximo o que já existe ([09:30] Larissa):

1. **Estrutura de módulo:** `src/modules/webhooks/` (novo) com controller, service, repository, routes e schemas, como os outros domínios ([09:27] Bruno). O processamento do worker também fica dentro do módulo ([09:28] Bruno).
2. **Erros:** classes novas estendem `AppError` e usam códigos com prefixo **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa).
3. **Tratamento de erros:** o error middleware centralizado trata os erros do módulo sem alteração ([09:29] Bruno).
4. **Logger:** Pino, sem biblioteca nova de log ([09:29] Bruno).
5. **Autorização:** o endpoint de replay da DLQ usa o `requireRole('ADMIN')` existente ([09:36] Larissa).
6. **Identificadores:** UUID em todas as tabelas novas, incluindo a outbox, "segue o padrão do resto do projeto" ([09:51] Larissa).
7. **Banco:** a stack Prisma + MySQL é a mesma. O worker usa um PrismaClient próprio com a mesma `DATABASE_URL` ([09:30] Bruno).

## Alternativas Consideradas

### A1. Estrutura e bibliotecas próprias para o módulo de webhooks (alternativa plausível, descartada)

Exemplos: um logger dedicado, formato de erro próprio, IDs auto-incrementais na outbox.

- **Prós:** liberdade para otimizar, como um ID sequencial para ordenação natural da outbox.
- **Contras:** fragmenta a codebase, exige mudar o error middleware e quebra a previsibilidade para quem já conhece o projeto. Diego chegou a perguntar sobre auto-incremento x UUID na outbox, e a decisão foi UUID pelo padrão do projeto ([09:51] Diego, [09:51] Larissa).

### A2. Compartilhar a instância do PrismaClient da API com o worker (descartada)

- **Contras:** inviável, porque o PrismaClient é por processo e o worker é outro processo Node ([09:30] Bruno).

## Consequências

### Positivas

- Curva de aprendizado mínima para o time, já que o módulo "fica como módulo igual aos outros" ([09:30] Larissa).
- O error middleware, o `validate` e o `requireRole` funcionam sem modificação.
- Respostas de erro com o mesmo envelope `{ error: { code, message, details } }` da API inteira.

### Negativas

- `NotFoundError` em `src/shared/errors/http-errors.ts` fixa o código `NOT_FOUND`. Para ter `WEBHOOK_NOT_FOUND`, as classes do módulo precisam estender `AppError` diretamente, sem herdar das classes HTTP genéricas.
- O `validate` middleware converte todo `ZodError` em `VALIDATION_ERROR`. Para que a URL `http` gere `WEBHOOK_INVALID_URL`, a validação de `https` precisa ser feita no service com um schema Zod dedicado (detalhado no FDD).
- Com UUID na outbox, a ordenação depende de `created_at`, não do id.

### Trade-off explícito

Abrimos mão de **otimizações locais**, como ID sequencial ou um formato de erro próprio, em troca de **consistência da codebase e zero mudança na infraestrutura compartilhada**.
