# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

> O enunciado original deste desafio está no histórico do repositório (commit `e7f6311`, arquivo `README.md`). Este README documenta **o processo** de produção da entrega.

## Sobre o desafio

Uma empresa opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) e decidiu, em uma reunião de ~55 minutos, construir um **Sistema de Webhooks de Notificação de Pedidos**. A única coisa registrada foi a transcrição da call (`TRANSCRICAO.md`). O desafio foi transformar essa conversa e o código existente em um pacote de design docs acionável: PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade. Nada podia ser inventado: cada requisito, decisão ou restrição precisava apontar para uma fala da reunião ou para um arquivo do código.

A parte difícil não foi escrever, foi **filtrar**. A reunião mistura decisões fechadas, propostas descartadas, pontos adiados, detalhes secundários e até ambiguidades numéricas. Também foi preciso manter cada documento na sua "altura": o PRD fala de produto, o RFC de arquitetura, os ADRs de decisões pontuais e o FDD de implementação, sem repetir conteúdo entre eles.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code (Claude Opus 5.5), no app desktop da Claude** | Ferramenta principal. Leu a transcrição e o código, extraiu e classificou os itens, redigiu todos os documentos e executou scripts de validação sobre a própria entrega |
| **Scripts Python gerados pela IA** (executados localmente) | Validação automática do Tracker: cada `[hh:mm] Nome` existe na transcrição? Cada caminho `CODIGO` existe no repo? Proporção TRANSCRICAO x CODIGO? IDs citados nos docs têm linha no Tracker? |

## Workflow adotado

1. **Contexto antes de requisitos.** Passei o enunciado em três blocos (contexto, requisitos, critérios de aceite) e pedi que a IA só começasse depois de receber os três, para que ela não otimizasse cedo demais para um critério parcial.
2. **Leitura integral das fontes.** A IA leu a transcrição inteira e os arquivos centrais do código: `prisma/schema.prisma`, `src/modules/orders/*`, `src/shared/errors/*`, `src/middlewares/*`, `src/config/*`, `src/server.ts`, `src/app.ts`, `src/routes/index.ts`, `package.json` e testes.
3. **Extração dirigida.** Antes de redigir, a IA montou um inventário classificando cada fala como *decisão fechada*, *alternativa descartada*, *adiado/fora de escopo*, *questão em aberto*, *requisito funcional*, *requisito não funcional* ou *gancho com o código* (prompt 1 abaixo).
4. **Ordem de produção, de baixo para cima nas decisões:**
   1. **ADRs**, porque as decisões são a espinha. Os 6 temas obrigatórios viraram 6 ADRs e o snapshot de payload ganhou um ADR-007.
   2. **PRD**, com o porquê e o quê, linkando os ADRs em vez de repetir o raciocínio.
   3. **RFC**, conciso, com alternativas e questões em aberto, apontando para o FDD em vez de detalhar.
   4. **FDD**, a parte mais longa: modelo de dados, fluxos, contratos, erros, integração com o código.
   5. **Tracker**, montado a partir dos IDs que cada documento já usa (`PRD-FR-01`, `RFC-ALT-02`, `FDD-INT-03`…).
5. **Validação automatizada + revisão crítica.** Um script cruzou o Tracker e os documentos com a transcrição e o sistema de arquivos (prompt 3). As divergências encontradas foram corrigidas.

## Prompts customizados

**Prompt 1: extração dirigida (filtrar o que entra e o que NÃO entra)**

```text
Leia TRANSCRICAO.md inteira. Não resuma. Produza um inventário em tabela com as colunas:
timestamp | falante | trecho | classificação | observação.

Classificações permitidas (use exatamente uma):
- DECISÃO FECHADA (alguém disse "decidido", "anotado", "tá fechado" ou o resumo final confirmou)
- ALTERNATIVA DESCARTADA (proposta colocada na mesa e rejeitada; registre o motivo)
- ADIADO / FORA DE ESCOPO (explicitamente empurrado para depois ou negado)
- QUESTÃO EM ABERTO (levantada e não decidida)
- REQUISITO FUNCIONAL / REQUISITO NÃO FUNCIONAL
- GANCHO COM O CÓDIGO (menção a arquivo, classe, método, padrão existente)

Regras:
- Se um valor numérico for citado mais de uma vez, confira se as menções são coerentes
  entre si e sinalize qualquer contradição.
- Uma sugestão sem confirmação de outro participante NÃO é decisão.
- Para cada GANCHO COM O CÓDIGO, abra o arquivo citado e confirme que ele existe e
  que o comportamento descrito na fala bate com o código.
```

**Prompt 2: fronteira entre documentos (evitar duplicação)**

```text
Antes de escrever, para cada item do inventário diga em qual documento ele MORA e em
quais ele é apenas REFERENCIADO por link:
- PRD: problema, público, escopo, métricas, requisitos (o quê / por quê). Sem nomes de tabela.
- RFC (2 a 4 páginas): abordagem, alternativas com trade-off, questões em aberto, riscos
  arquiteturais. Sem payloads, sem SQL, sem matriz de erros.
- ADR: uma decisão por arquivo, no formato MADR (Status, Contexto, Decisão, Alternativas
  Consideradas, Consequências com trade-off explícito).
- FDD: modelo de dados, fluxos, contratos HTTP com exemplos, erros WEBHOOK_*, resiliência,
  observabilidade, integração com arquivos reais do código.
Todo detalhe de implementação que NÃO foi decidido na reunião deve ser marcado como
"(proposta FDD)" e listado numa seção própria, com a fala ou o arquivo de que ele deriva.
Todo arquivo que ainda não existe deve ser marcado como "(novo)".
```

**Prompt 3: auditoria anti-alucinação do Tracker**

```text
Escreva e execute um script que:
1. extraia todos os pares (hh:mm, falante) reais de TRANSCRICAO.md;
2. para cada linha do docs/TRACKER.md com Fonte=TRANSCRICAO, verifique se a Localização
   existe exatamente nesse conjunto; com Fonte=CODIGO, verifique se o caminho existe;
3. calcule o % de linhas TRANSCRICAO e o número de linhas CODIGO;
4. liste todo `[hh:mm] Nome` citado em PRD/RFC/FDD/ADRs que não exista na transcrição;
5. liste todo caminho `src/...`, `prisma/...`, `tests/...` citado nos docs que não exista
   no repositório e não esteja marcado como (novo);
6. liste todo ID (PRD-*, RFC-*, FDD-*) usado nos docs que não tenha linha no Tracker.
Depois corrija cada divergência na fonte, sem relaxar o script.
```

## Iterações e ajustes

Foram **4 iterações principais**: (1) extração e classificação, (2) redação dos documentos, (3) validação automatizada e correção, (4) auditoria final contra as "Dicas Finais" do enunciado. Nas iterações 2 e 3, estes foram os pontos em que a primeira versão estava errada ou superficial:

1. **Ambiguidade das "5 tentativas".** A primeira leitura tratava "5 tentativas" como 5 envios no total. A conta não fecha: com 5 envios só cabem 4 intervalos (1m+5m+30m+2h ≈ 2h36), e o Diego fala em "quase 15 horas" usando os 5 intervalos (≈ 14h36). Ajuste: o FDD adota "1 envio inicial + 5 retentativas", e a ambiguidade virou a questão em aberto `RFC-OQ-07` em vez de ser resolvida em silêncio.
2. **Conflito entre o código de erro e o middleware real.** O rascunho dizia que a URL `http` seria recusada "no schema Zod com `WEBHOOK_INVALID_URL`". Lendo `src/middlewares/validate.middleware.ts`, todo `ZodError` vira `VALIDATION_ERROR`. Lendo `src/shared/errors/http-errors.ts`, `NotFoundError` fixa o código `NOT_FOUND`. Ajuste: a regra https fica num schema Zod dedicado, executado no service, e as classes `Webhook*Error` estendem `AppError` diretamente (ADR-006, FDD-INT-03 e FDD-INT-05).
3. **Limite de 64KB no lugar errado.** A primeira versão validava o tamanho na inserção da outbox. Isso faria a **mudança de status do pedido dar rollback** por causa do webhook, o oposto do que a reunião quis proteger. Ajuste: a validação passou para o worker, com o evento indo direto para a DLQ (FDD-FLX-12).
4. **Ordem quebrada por retry.** "Single-worker processando por `created_at`" não garante ordem quando um evento antigo está em backoff e um novo do mesmo pedido fica pendente. Ajuste: a regra de elegibilidade por (webhook, pedido) foi marcada como proposta (FDD-FLX-07 / FDD-IMP-01), com o risco de head-of-line blocking documentado.
5. **FK em cascata apagaria eventos.** O rascunho do modelo tinha FK `orderId → orders` com cascade. `OrderService.delete` permite apagar pedidos `CANCELLED`, o que apagaria o próprio evento de cancelamento ainda pendente. Ajuste: `orderId` ficou sem FK (FDD-DADOS-04).
6. **Bytes assinados instáveis.** O payload estava como coluna `JSON`. O MySQL reordena chaves de JSON, e o HMAC precisa dos mesmos bytes em todo retry. Ajuste: o payload é guardado como texto serializado (FDD-DADOS-05).
7. **Achados da validação automatizada.** O script apontou arquivos que ainda não existem (`src/worker.ts`, `src/modules/webhooks/`) citados sem a marcação "(novo)" no ADR-006 e no Tracker, o que feriria o critério "nenhum arquivo mencionado é inexistente". Ajuste: tudo marcado. O script também confirmou que todos os timestamps citados existem e que ~86% das linhas do Tracker são `TRANSCRICAO`.
8. **Auditoria contra as Dicas Finais (iteração 4).** Procurei itens descartados vazando como requisito (e-mail, dashboard, rate limit, inbound, arquivamento): nenhum, todos só em "fora de escopo" ou "em aberto". A revisão linha a linha do Tracker, porém, achou quatro problemas:
   - **Meta contraditória.** O PRD-OBJ-01 fixava meta < 10 s, mas a justificativa somava 2 s de polling + 10 s de timeout = 12 s.
   - **Origem fraca no Tracker.** RFC-META-01 apontava para uma fala que não tratava de revisores.
   - **Códigos de erro sem origem marcada.** Só três códigos `WEBHOOK_*` foram citados na reunião; os outros nove, derivados do prefixo, apareciam como se tivessem sido falados.
   - **Detalhes inventados sem marcação.** O nome da rota de rotação e a ordem de deploy não foram definidos na reunião.

   Ajuste: meta reescrita, origem corrigida e os itens derivados marcados como **(proposta FDD)** (FDD-IMP-10 e FDD-IMP-11).

## Como navegar a entrega

Ordem de leitura sugerida:

| # | Arquivo | Para quê |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](docs/PRD.md) | Problema, clientes, escopo (incluindo o que ficou de fora), requisitos e métricas |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta de arquitetura, alternativas descartadas e questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/README.md) | As 7 decisões, uma por arquivo (ADR-001 a ADR-007) |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: dados, fluxos, contratos, erros, observabilidade, integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item (fala da reunião ou arquivo do código) |
| — | [`TRANSCRICAO.md`](TRANSCRICAO.md) | Fonte primária |

```
docs/
├── PRD.md
├── RFC.md
├── FDD.md
├── TRACKER.md
└── adrs/
    ├── README.md
    ├── ADR-001-outbox-no-mysql.md
    ├── ADR-002-worker-separado-em-polling.md
    ├── ADR-003-retry-com-backoff-exponencial-e-dlq.md
    ├── ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md
    ├── ADR-005-entrega-at-least-once-com-x-event-id.md
    ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
    └── ADR-007-payload-snapshot-enxuto-na-insercao.md
```

O código da aplicação (`src/`, `prisma/`, `tests/`, configurações) **não foi alterado**. Ele serviu apenas como fonte de contexto.
