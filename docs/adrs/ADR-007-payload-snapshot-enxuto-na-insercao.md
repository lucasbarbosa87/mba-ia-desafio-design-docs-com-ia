# ADR-007: Payload enxuto renderizado como snapshot na inserção da outbox, com filtro de eventos na inserção

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisões em [09:34], [09:43] e [09:52])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Consultados:** Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)

## Contexto

Ao inserir um evento na outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), três perguntas precisam de resposta:

1. **O que** guardar: o payload pronto ou só o `order_id` para renderizar no envio ([09:51] Bruno)?
2. **Quanto** enviar: o pedido completo com itens ou só os campos essenciais ([09:43] Marcos)?
3. **Quando** aplicar o filtro de status que cada webhook quer ouvir: na inserção ou no envio ([09:34] Diego)?

## Decisão

1. **Snapshot na inserção:** o payload é renderizado e gravado na outbox no momento da mudança de status. Se o pedido mudar depois, o evento ainda reflete o estado do momento da transição ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
2. **Payload enxuto em JSON** ([09:43] Diego):
   - **Campos:** `event_id`, `event_type` (`"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`.
   - **Sem `items`:** o payload não inclui `items`. Quem precisar de detalhes consulta `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).
3. **Filtro na inserção:** só é inserida linha para webhooks ativos do customer que assinam o `to_status`. Se nenhum quer aquele status, nada é inserido ([09:34] Bruno, [09:34] Diego).
4. **Limite de 64KB:** um evento acima do limite **não é enviado** e é tratado como erro, sem truncamento ([09:23] Sofia, [09:24] Larissa). É um requisito não funcional, não uma decisão arquitetural à parte ([09:24] Larissa).

## Alternativas Consideradas

### A1. Guardar só `order_id` e renderizar no envio (descartada)

- **Prós:** linha menor na outbox e dados sempre "frescos".
- **Contras:** se o pedido mudar entre a inserção e o envio, o evento passa a descrever um estado diferente do que gerou a notificação ("tem caso esquisito", [09:52] Larissa). Também acopla o worker à leitura de pedidos e a retentativa enviaria payloads diferentes.

### A2. Enviar o pedido completo, com itens (descartada)

- **Contras:** infla o payload sem necessidade ([09:43] Diego). Quem precisa de mais detalhes já tem o `GET /orders/:id`.

### A3. Filtrar no momento do envio (descartada)

- **Contras:** insere linhas que nunca serão enviadas. Filtrar na inserção "economiza linha na tabela" ([09:34] Bruno).

## Consequências

### Positivas

- Payload estável: retentativas e replays enviam exatamente o mesmo corpo, com a mesma assinatura HMAC sobre os mesmos bytes ([ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)).
- Payload pequeno, muito abaixo do teto de 64KB ([09:24] Diego).
- Outbox sem linhas inúteis.

### Negativas

- **Filtro congelado na inserção:** uma alteração do filtro de eventos (PATCH) não afeta eventos já inseridos.
- **Leitura extra no `changeStatus`:** a transação passa a buscar os webhooks ativos do customer.
- **Detalhes exigem nova chamada:** o cliente que precisa de itens faz uma chamada adicional à API.

### Trade-off explícito

Escolhemos **fidelidade ao momento do evento e payload pequeno** em vez de **dados sempre atualizados e completos**, empurrando o detalhamento para o `GET /orders/:id` existente.
