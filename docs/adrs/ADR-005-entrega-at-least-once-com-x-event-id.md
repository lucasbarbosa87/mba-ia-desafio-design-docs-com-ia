# ADR-005: Garantia de entrega at-least-once com deduplicação pelo cliente via `X-Event-Id`

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão fechada em [09:26])
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma)
- **Consultados:** Sofia (Segurança), Marcos (PM), Bruno (Pedidos)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Com outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) e retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), há cenários em que o cliente recebe o mesmo evento mais de uma vez. Dois exemplos:

- o cliente processou, mas a resposta não chegou antes do timeout de 10 s;
- o worker caiu depois de enviar e antes de marcar como entregue.

Precisamos definir qual garantia de entrega oferecemos e como o cliente lida com duplicatas.

## Decisão

1. **Garantia at-least-once:** o cliente pode receber o mesmo evento duas vezes e precisa estar preparado ([09:24] Diego).
2. **Header `X-Event-Id`:** cada evento leva um UUID **gerado no momento em que entra na outbox** e único por evento. O cliente deduplica por esse id ([09:25] Diego).
   - O id não muda entre retentativas.
   - Também não muda num replay vindo da DLQ, já que o replay recoloca **o mesmo evento** na outbox ([09:18] Diego).
3. **Header `X-Webhook-Id`:** cada envio leva o id do endpoint cadastrado. Assim, um cliente com vários cadastros sabe qual deles originou o envio ([09:44] Sofia, [09:45] Diego).
4. **Portal do desenvolvedor:** a necessidade de deduplicar é documentada com destaque no portal ([09:26] Marcos).

## Alternativas Consideradas

### A1. Exactly-once (descartada)

- **Prós:** o cliente não precisaria deduplicar.
- **Contras:** exigiria coordenação dos dois lados e "fica muito mais complexo" ([09:25] Diego). É impraticável sobre HTTP com clientes heterogêneos.

### A2. At-most-once, sem retry (alternativa plausível, descartada)

- **Prós:** nunca há duplicata.
- **Contras:** qualquer instabilidade do cliente vira perda de evento. Isso contradiz a política de retry de [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) e o objetivo de não deixar o cliente "pendurado" ([09:02] Marcos).

## Consequências

### Positivas

- É o padrão de mercado ("Stripe faz assim, GitHub faz assim", [09:25] Diego) e por isso familiar para os integradores.
- Implementação simples do nosso lado: na dúvida, reenviar.
- Resolve a grande maioria dos casos ([09:25] Diego).

### Negativas

- Joga responsabilidade para o cliente ([09:25] Sofia). Um cliente que não deduplica pode processar a mesma mudança de status duas vezes.
- Exige documentação clara e suporte ao integrador.

### Trade-off explícito

Preferimos **simplicidade e confiabilidade de entrega do nosso lado** a **conveniência total para o cliente**. Assumimos o custo de documentar e orientar a deduplicação.
