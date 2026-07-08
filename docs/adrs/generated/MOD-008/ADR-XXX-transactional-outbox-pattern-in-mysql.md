# ADR-XXX — Uso do Padrão Transactional Outbox no MySQL para Captura Atômica de Eventos de Webhook

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Diego (Engenheiro Sênior, time de Plataforma), Bruno (Engenheiro Pleno, time de Pedidos), Larissa (Tech Lead)
- **Revisores:** Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

O sistema possui três clientes B2B estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — que necessitam ser notificados em tempo real quando o status de seus pedidos é alterado na plataforma. A situação atual obriga esses clientes a realizar polling constante via `GET /orders`, gerando latência na integração e custo operacional desnecessário. A Atlas Comercial sinalizou formalmente que, caso a solução não seja entregue até o fim do trimestre, poderá migrar para um concorrente (`[09:00] Marcos`).

O ponto crítico de integração é o método `OrderService.changeStatus` (arquivo `src/modules/orders/order.service.ts`), que já executa dentro de um bloco `prisma.$transaction(async (tx) => { ... })`. Essa transação é responsável por atualizar o status do pedido na tabela `orders`, registrar a entrada em `order_status_history` e, quando aplicável, decrementar ou repor o `stockQuantity` dos produtos envolvidos nos itens do pedido. Toda a persistência do sistema utiliza MySQL como banco de dados, conforme declarado no `prisma/schema.prisma` (`datasource db { provider = "mysql" ... }`), e o acesso ao banco é feito exclusivamente via Prisma ORM com o tipo `Prisma.TransactionClient` (alias `TxClient`) já amplamente utilizado no código.

Diante desse cenário, o time precisava de um mecanismo que garantisse a captura durável do evento de notificação de forma atômica com a mudança de status, sem introduzir nova infraestrutura e sem bloquear a transação principal com chamadas HTTP síncronas a endpoints externos de disponibilidade incerta.

---

## Decisão

O time decidiu adotar o **Transactional Outbox Pattern** sobre o MySQL existente (`[09:06] Diego`, `[09:07] Larissa`, `[09:08] Larissa`). A decisão central é: quando `OrderService.changeStatus` realiza uma transição de status, uma chamada à função `publishWebhookEvent(tx, order, fromStatus, toStatus)` é feita dentro do mesmo bloco `prisma.$transaction`. Essa função insere uma linha na tabela `webhook_outbox` usando o `Prisma.TransactionClient` (`tx`) recebido como parâmetro, garantindo que o evento seja persistido se e somente se a mudança de status for commitada — e descartado automaticamente caso haja rollback (`[09:40–09:41] Bruno, Diego`).

O payload do evento é serializado e armazenado como snapshot no momento da inserção na outbox, refletindo o estado exato da order no instante da transição. Isso evita inconsistências caso o pedido seja modificado depois da mudança de status (`[09:52] Larissa, Diego`). Um worker separado (`src/worker.ts`, com a lógica encapsulada em `src/modules/webhooks/webhook.processor.ts` ou `webhook.worker.ts`) realiza polling a cada 2 segundos na tabela `webhook_outbox`, lê os eventos com status pendente em batches, despacha as chamadas HTTP aos endpoints dos clientes e atualiza o status do registro para entregue — ou inicia o ciclo de retry em caso de falha (`[09:09] Diego`). O worker roda como processo Node.js separado da API, compartilhando a mesma `DATABASE_URL` mas instanciando um `PrismaClient` próprio (`[09:11] Diego`, `[09:30] Bruno`).

---

## Alternativas Consideradas

### Chamada HTTP síncrona dentro de `changeStatus`

A abordagem mais imediata seria chamar diretamente o endpoint do cliente B2B dentro do método `changeStatus`, antes ou após o commit da transação. Essa alternativa foi rejeitada porque um endpoint de cliente lento ou indisponível bloquearia a execução da transação inteira de mudança de status, causando timeouts e impacto para outros pedidos. Além disso, se o cliente estiver fora do ar, seria necessário ou ignorar o evento (perda de notificação) ou fazer rollback da mudança de status (comportamento inaceitável). (`[09:04] Bruno`, `[09:06] Diego`)

### Redis Streams / Redis Cluster

Redis Streams foi levantado como alternativa mais sofisticada para publicação de eventos de forma desacoplada. A alternativa foi rejeitada porque exigiria a criação e operação de infraestrutura nova — um Redis Cluster — para a qual o time não possui experiência operacional. Para um time pequeno, isso representa risco operacional e custo desproporcionais ao problema a ser resolvido, sendo classificado como overengineering dado o que já existe disponível. (`[09:07–09:08] Diego, Larissa`)

### Trigger de banco de dados MySQL

O uso de triggers MySQL foi considerado como alternativa para detectar mudanças na tabela `orders` de forma reativa, sem alteração no código da aplicação. A alternativa foi rejeitada porque triggers MySQL não possuem mecanismo nativo equivalente ao `NOTIFY/LISTEN` do PostgreSQL para notificar processos externos. Qualquer tentativa de acionar o worker a partir de uma trigger exigiria soluções improvisadas (escrita em arquivo, chamada a endpoint interno) que introduziriam efeitos colaterais opacos e invisíveis ao código da aplicação. (`[09:09] Bruno, Diego`)

---

## Consequências

### Positivas

- **Atomicidade garantida:** o evento é registrado em `webhook_outbox` dentro da mesma transação SQL que persiste a mudança de status. Não existe estado intermediário em que o status mudou mas o evento não foi capturado, e vice-versa.
- **Sem nova infraestrutura:** a solução reutiliza a instância MySQL já existente e o Prisma ORM já configurado no projeto, sem adicionar Redis, Kafka, RabbitMQ ou qualquer outro componente externo.
- **Integração limpa com o código atual:** a função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o `Prisma.TransactionClient` já disponível em `changeStatus`, mantendo o contrato de tipos existente (`TxClient`) sem necessidade de refatorações extensas no `OrderService`.
- **Desacoplamento do ciclo de requisição:** o worker processa e despacha os eventos no seu próprio ritmo, sem bloquear ou ser bloqueado pelo ciclo de request/response da API.
- **Snapshot preserva fidelidade histórica:** ao serializar o payload no momento da inserção na outbox, o evento sempre refletirá o estado exato da order no instante da transição, independentemente de alterações posteriores no pedido.
- **Atende ao requisito de latência:** o polling de 2 segundos garante entrega dentro da janela de "abaixo de 10 segundos" exigida pelos clientes B2B (`[09:10] Marcos`).
- **Ordering por pedido preservada:** com um único worker processando por `created_at`, eventos do mesmo pedido são entregues na ordem correta de transição de status.

### Negativas / Trade-offs

- **Crescimento da tabela `webhook_outbox`:** registros entregues acumulam na tabela e exigem política de limpeza periódica (arquivamento ou deleção de registros com status `delivered` após janela de retenção, ex.: 30 dias). Sem essa rotina, a tabela pode crescer indefinidamente e degradar a performance do índice.
- **Latência mínima de 2 segundos:** o polling introduz uma latência mínima de 2 segundos entre a mudança de status e o início do envio do webhook. Não é entrega instantânea, embora atenda ao SLA acordado.
- **At-least-once delivery:** em cenários de falha e retry, o cliente pode receber o mesmo evento mais de uma vez. A responsabilidade de deduplicação recai sobre o cliente, que deve implementar idempotência usando o `X-Event-Id` (UUID gerado na inserção da outbox) enviado em cada requisição (`[09:25] Diego`).
- **Escalabilidade limitada do MySQL como broker de eventos:** o MySQL não é otimizado para cenários de alta taxa de eventos. Se o volume de mudanças de status crescer significativamente, o design do outbox sobre MySQL pode precisar ser revisado em favor de uma solução de mensageria dedicada.
- **Ordering não garantida com múltiplos workers:** a garantia de ordering por `order_id` é válida apenas enquanto o sistema operar com um único worker. Escalar para múltiplos workers paralelos exigiria particionamento por `order_id` ou uso de locks pessimistas (`[09:12–09:13] Diego`). Isso está documentado como limitação conhecida.
- **Complexidade operacional do ciclo de vida do evento:** a tabela `webhook_outbox` requer modelagem de estados (pendente, processando, falhou, entregue), índices adequados em status e `created_at`, e integração com a tabela `webhook_dead_letter` para eventos que esgotaram todas as tentativas de retry (`[09:08] Diego`, `[09:18] Diego`).
- **Processo separado a operar e monitorar:** o worker (`src/worker.ts`) precisa de supervisão, reinicialização automática em caso de crash e monitoramento independente da API, adicionando complexidade operacional ao deploy.

---

## Referências

### Transcrição

- `[09:00] Marcos` — contexto de negócio: pedido formal dos clientes B2B Atlas Comercial, MaxDistribuição e Nova Cargo
- `[09:03] Larissa` — levantamento inicial: síncrono vs. fila/outbox
- `[09:04] Bruno` — rejeição da abordagem síncrona: peso da transação atual e risco de rollback indevido
- `[09:06] Diego` — definição do padrão outbox e descrição da mecânica de inserção atômica na `webhook_outbox`
- `[09:07] Larissa` — menção a Redis Streams como alternativa e rejeição por custo de infra
- `[09:07–09:08] Diego` — rejeição de Redis Cluster (overengineering para time pequeno) e confirmação do outbox no MySQL existente
- `[09:09] Bruno, Diego` — rejeição de triggers MySQL; definição de polling a cada 2 segundos
- `[09:10] Marcos` — confirmação de que 2 segundos atende ao requisito de "abaixo de 10 segundos"
- `[09:11] Diego` — worker como processo separado da API
- `[09:12–09:13] Diego` — limitação de ordering com múltiplos workers; documentação como limitação conhecida
- `[09:25] Diego` — at-least-once com `X-Event-Id` para deduplicação pelo cliente
- `[09:40–09:41] Bruno, Diego` — integração crítica: `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação de `changeStatus`
- `[09:51] Larissa` — UUID como identificador da outbox, seguindo padrão do restante do projeto
- `[09:52] Larissa, Diego, Bruno` — decisão de serializar payload como snapshot na inserção

### Código

- `src/modules/orders/order.service.ts` — método `changeStatus` com `prisma.$transaction(async (tx) => { ... })`; tipo `TxClient = Prisma.TransactionClient`; métodos `debitStock` e `replenishStock` chamados dentro da transação
- `prisma/schema.prisma` — datasource MySQL; modelos `Order`, `OrderStatusHistory`, `OrderItem`, `OrderNumberSequence`; enum `OrderStatus` (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED)
- `docs/adrs/potential-adrs/must-document/MOD-008/transactional-outbox-in-mysql.md` — evidências do potential ADR de origem (score 150/150)

### Módulos Relacionados

- **MOD-007 (Orders):** ponto de integração é `OrderService.changeStatus` em `src/modules/orders/order.service.ts`
- **MOD-008 (Webhooks):** proprietário da tabela `webhook_outbox`, da tabela `webhook_dead_letter`, do worker e do processor
- **MOD-001 (Infrastructure):** `prisma/schema.prisma` receberá os novos modelos de outbox e dead letter
