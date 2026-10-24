# ADR-004: Assinatura HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h

- **Status:** Aceito (sujeito à revisão de segurança antes do deploy, [09:46] Sofia)
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão fechada em [09:22])
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead)
- **Consultados:** Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-snapshot-enxuto-na-insercao.md)

## Contexto

Os webhooks levam dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir validar duas coisas ([09:19] Sofia):

- que a requisição veio de nós;
- que o payload não foi adulterado no caminho.

Já houve caso de cliente que vazou secret em log da própria aplicação ([09:22] Diego). O fluxo é só outbound: nós enviamos, o cliente não envia para nós ([09:02] Marcos, [09:03] Sofia).

## Decisão

1. **Assinatura:** assinar o **corpo do request** com HMAC-SHA256 e enviar a assinatura no header `X-Signature` ([09:20] Sofia, [09:22] Sofia).
2. **Secret por endpoint:** cada webhook cadastrado tem sua própria secret, sem secret global da plataforma ([09:21] Sofia).
   - A configuração armazena `url`, `secret`, `customer_id` e o estado ativo ([09:21] Bruno).
   - A secret é **gerada pela plataforma** e devolvida ao cliente na criação ([09:31] Marcos).
3. **Rotação:** o cliente pode pedir uma nova secret pela API. A secret antiga continua válida por **24 horas** em paralelo e depois é descartada ([09:21] Sofia).
4. **TLS obrigatório:** a URL precisa ser `https`. URLs `http` são recusadas com erro de validação no schema Zod ([09:23] Sofia).
5. **Revisão de segurança:** Sofia revisa o código de HMAC e de geração de secret antes do deploy, com pelo menos 2 dias úteis reservados ([09:46] Sofia).

## Alternativas Consideradas

### A1. Secret global única da plataforma (descartada)

- **Prós:** gestão trivial, uma única chave.
- **Contras:** se vaza uma, vaza tudo ([09:21] Sofia). Um único vazamento comprometeria a autenticidade dos envios para todos os clientes.

### A2. Rotação sem período de convivência (alternativa plausível, descartada)

- **Prós:** a secret antiga morre na hora, com menor janela de exposição.
- **Contras:** os eventos assinados com a secret nova seriam rejeitados até o cliente atualizar todos os seus sistemas. O grace period de 24h existe justamente para dar tempo de migração ([09:21] Sofia).

## Consequências

### Positivas

- HMAC-SHA256 é padrão de mercado e qualquer cliente sério tem biblioteca para verificar ([09:20] Sofia).
- O vazamento de uma secret afeta um único endpoint e pode ser contido por rotação.
- `https` obrigatório protege confidencialidade e integridade em trânsito.

### Negativas

- **Secret em forma recuperável:** a secret precisa ser armazenada de forma recuperável, porque o worker precisa dela para assinar. Ao contrário de uma senha, não dá para guardar só o hash. A proteção em repouso é ponto de atenção da revisão de segurança.
- **Duas secrets na rotação:** durante as 24h o worker lida com duas secrets por endpoint, o que adiciona complexidade de modelo (`secret` + `previous_secret` com expiração).
- **Verificação do lado do cliente:** a responsabilidade de verificar a assinatura é do cliente. Se ele não verifica, a proteção não tem efeito.

### Trade-off explícito

Aceitamos **mais complexidade de gestão de secrets** (uma por endpoint, com rotação e janela dupla) em troca de **isolamento de vazamentos** e de **rotação sem downtime para o cliente**.
