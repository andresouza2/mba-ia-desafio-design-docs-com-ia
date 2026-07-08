# ADR-XXX — Payload da Outbox Armazenado como Snapshot no Momento da Inserção

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, time de Plataforma), Bruno (Engenheiro Pleno, time de Pedidos)
- **Revisores:** Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Relações

- **Depende de:** [ADR-001 — Uso do Padrão Transactional Outbox no MySQL para Captura Atômica de Eventos de Webhook](./ADR-001-transactional-outbox-pattern-in-mysql.md)
- **Altera:** [ADR-003 — Worker de Webhooks com Polling de 2 Segundos na webhook_outbox](./ADR-003-worker-polling-intervalo-dois-segundos.md)
- **Relacionado a:** [ADR-006 — Garantia de Entrega At-Least-Once com Deduplicação via X-Event-Id](./ADR-006-garantia-at-least-once-com-x-event-id.md)

---

## Contexto

O módulo de webhooks (MOD-008) utiliza o padrão _outbox_ para garantir a entrega confiável de notificações aos clientes B2B quando o status de um pedido muda. A tabela `webhook_outbox` recebe uma nova linha dentro da mesma transação SQL que executa a atualização de status em `orders`, o registro em `order_status_history` e o ajuste de `stock_quantity` dos produtos — garantindo atomicidade total entre a mudança de negócio e o registro do evento a ser entregue.

Um ponto de decisão crítico surgiu ao final da reunião técnica: a coluna `payload` da `webhook_outbox` deve armazenar o JSON completo do evento já renderizado no momento da inserção, ou apenas o `order_id`, delegando a renderização ao worker no momento em que ele for efetivamente disparar a requisição HTTP ao endpoint do cliente (estratégia de _lazy rendering_)? A distinção importa porque esses dois momentos podem ser separados por um intervalo significativo de tempo: o worker adota backoff exponencial com até 5 tentativas nos intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas — somando uma janela de aproximadamente 15 horas entre a primeira falha de entrega e a última tentativa. Durante esse intervalo, o pedido pode sofrer mutações legítimas como correção de endereço, ajuste de itens ou novas transições de status.

O método `changeStatus` em `src/modules/orders/order.service.ts` já executa um `tx.order.findUnique` ao final da transação (linhas 169–176) para refazer a leitura do pedido com todas as suas relações (`items`, `history`, `customer`) e retornar o objeto atualizado ao chamador. Esse objeto re-fetchado, que reflete exatamente o estado do pedido no instante da transição, é a fonte natural para a construção do snapshot de payload. A coluna `webhook_outbox.payload` é definida como `Json` no schema Prisma, comportando o documento completo sem restrições de formato adicional.

---

## Decisão

A função `publishWebhookEvent(tx, order, fromStatus, toStatus)` — proposta por Bruno em `[09:41]` e aprovada por Diego e Larissa — recebe o objeto completo do pedido (incluindo as relações carregadas) e serializa imediatamente o payload JSON na linha que está sendo inserida em `webhook_outbox`. O payload inclui `event_id` (UUID gerado na inserção), `event_type` (`"order.status_changed"`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e os campos principais do pedido como `total_cents`. Esse JSON é gravado na coluna `payload` (tipo `Json` no schema Prisma) no mesmo instante e dentro da mesma transação que registra a mudança de status.

O worker de entrega, ao processar cada linha da outbox, lê o campo `payload` já pronto e o utiliza diretamente no corpo da requisição HTTP sem realizar nenhuma consulta adicional ao módulo de pedidos. Isso torna o worker completamente _stateless_ em relação ao estado atual do pedido: ele apenas despacha o documento que foi capturado no momento da transição, assina com HMAC-SHA256 usando a secret do endpoint de destino, e registra o resultado em `webhook_deliveries`.

---

## Alternativas Consideradas

### Lazy rendering no momento do dispatch

Nessa abordagem, a `webhook_outbox` armazenaria apenas o `order_id` (e os metadados de transição `fromStatus`/`toStatus`), e o worker buscaria o estado atual do pedido via `GET /orders/:id` ou consulta direta ao banco no momento de cada tentativa de entrega.

A alternativa foi rejeitada pelos seguintes motivos, conforme deliberado em `[09:51]` por Larissa e confirmado em `[09:52]` por Diego e Bruno:

1. **Inconsistência de dados dentro da janela de retry:** Com backoff de até 15 horas (1m / 5m / 30m / 2h / 12h), o pedido pode ter sofrido atualizações após a transição original — por exemplo, correção de endereço de entrega, ajuste de desconto ou até uma nova mudança de status. O cliente receberia na quinta tentativa um payload diferente do que receberia na primeira, tornando impossível correlacionar o evento com a transição que o originou.

2. **Semântica incorreta do evento:** Um webhook `order.status_changed` de `PAID` para `PROCESSING` deve retratar o pedido como ele estava _quando essa transição ocorreu_, não como ele está _no momento do envio_. Usar o estado atual viola a semântica de evento e pode levar o sistema do cliente a tomar decisões erradas com base em dados que nunca foram o estado real no instante da transição.

3. **Acoplamento desnecessário ao módulo de pedidos:** O worker precisaria de acesso direto ao `OrderRepository` ou à API de pedidos para cada dispatch, criando dependência de runtime entre dois módulos que, com snapshot na inserção, permanecem desacoplados.

---

## Consequências

### Positivas

- **Determinismo total nas reentregas:** O payload é imutável após a inserção; todas as tentativas de retry enviam exatamente o mesmo JSON, independentemente de quantas mutações o pedido sofra depois. O cliente sempre recebe o estado do pedido no instante exato da transição.
- **Outbox como log de auditoria fiel:** Cada linha da `webhook_outbox` representa um registro preciso e reprodutível do estado do sistema no momento de cada transição de status, podendo ser usado para auditoria e diagnóstico histórico.
- **Worker stateless em relação ao domínio de pedidos:** O worker não precisa conhecer o `OrderRepository`, a lógica de relações do Prisma nem a estrutura de `OrderWithRelations`. Ele apenas lê, assina e despacha — reduzindo a superfície de acoplamento e facilitando testes unitários do worker.
- **Simplificação da lógica de retry:** Como o payload já está materializado, o tratamento de falhas de entrega se resume a retentar o mesmo HTTP call com o mesmo body, sem nenhum estado adicional a gerenciar.

### Negativas / Trade-offs

- **Tamanho da coluna `payload`:** Armazenar o documento JSON completo por evento aumenta o volume de dados na `webhook_outbox` proporcionalmente ao número de transições e à cardinalidade dos pedidos. Pedidos com muitos itens produzem payloads maiores. O limite de 64 KB discutido em `[09:23]` por Sofia e Diego serve como teto de segurança para evitar linhas excessivamente grandes.
- **`publishWebhookEvent` exige o objeto completo com relações:** A função não pode ser chamada com apenas o `order_id`; ela precisa receber o objeto `OrderWithRelations` já populado. No contexto de `changeStatus`, isso é naturalmente satisfeito pelo `findUnique` com `include` que já ocorre nas linhas 169–176 de `order.service.ts`, mas outros contextos de chamada futuros precisarão garantir o mesmo nível de hidratação do objeto.
- **Evolução de schema do payload:** Se a estrutura do evento JSON mudar (por exemplo, adição ou remoção de campos), linhas antigas na `webhook_outbox` conterão o formato anterior. O worker deve ser capaz de processar ambas as versões em deploys graduais (_rolling deployments_), o que exige atenção especial durante migrações de schema.
- **Sem reflexo de correções pós-transição:** Se um pedido tiver um dado incorreto no momento da transição de status (por exemplo, um endereço de entrega com erro de digitação que foi corrigido minutos depois), o payload entregue ao cliente via webhook refletirá o dado errado original. Isso é uma consequência direta e intencional do snapshot — a correção posterior não altera o evento já registrado.

---

## Referências

### Transcrição

- `[09:51] Larissa` — decisão de usar snapshot renderizado no momento da inserção: _"Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito."_
- `[09:52] Diego` — confirmação: _"Concordo, snapshot na inserção."_
- `[09:52] Bruno` — confirmação: _"Beleza, snapshot. Decidido."_
- `[09:41] Bruno` — proposta da assinatura da função: _"Vou propor uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que aceita o tx client da transação atual."_
- `[09:41] Diego` — aprovação do design: _"Boa, função pura recebendo o tx. Não precisa injetar repository inteiro."_
- `[09:17] Diego` — definição da janela de retry: _"1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa."_
- `[09:43] Diego` — definição do formato de payload: _"JSON com event_id, event_type tipo 'order.status_changed', timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id, e os campos básicos da order tipo total_cents."_
- `[09:23] Sofia` e `[09:24] Diego` — limite de 64 KB para o payload.

### Código

- `src/modules/orders/order.service.ts` — método `changeStatus` (linhas 126–179); o `tx.order.findUnique` com `include: { items, history, customer }` nas linhas 169–176 é o ponto onde o objeto `OrderWithRelations` está disponível para ser passado a `publishWebhookEvent`.
- `prisma/schema.prisma` — modelos `Order`, `OrderItem`, `Customer` e `OrderStatusHistory` definem a estrutura das relações que compõem o snapshot; a coluna `webhook_outbox.payload` será do tipo `Json`.
