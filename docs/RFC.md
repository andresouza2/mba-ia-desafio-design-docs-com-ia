# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| **Data** | 2026-07-06 |
| **Status** | Em revisão |

---

## Resumo Executivo (TL;DR)

Propomos substituir o polling que os clientes B2B fazem hoje em `GET /orders` por notificação ativa via **webhooks HTTP**. A captura do evento de mudança de status usa o **padrão Transactional Outbox sobre o MySQL já existente** (sem infraestrutura nova), inserido atomicamente dentro da mesma transação de `OrderService.changeStatus`. Um **worker Node.js separado**, em polling de 2 segundos, lê a outbox e entrega os eventos via HTTP, assinados com **HMAC-SHA256** e identificados por um **`X-Event-Id`** estável para deduplicação do lado do cliente (garantia **at-least-once**). Falhas de entrega seguem **retry com backoff exponencial** (5 tentativas, até ~15h) antes de irem para uma **Dead Letter Queue (DLQ)** com replay manual administrativo. O módulo `webhooks` reaproveita integralmente os padrões já estabelecidos no projeto (estrutura `controller/service/repository`, `AppError`, Pino, `requireRole`, error middleware).

---

## Contexto e Problema

Três clientes B2B estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente ser notificados em tempo real sobre mudanças de status em seus pedidos (`[09:00] Marcos`). Hoje eles fazem polling manual em `GET /orders`, o que é lento e caro para a integração deles; a Atlas sinalizou risco de migrar para um concorrente se a solução não for entregue até o fim do trimestre (`[09:00] Marcos`). O requisito de latência aceito pelos clientes é "abaixo de 10 segundos" (`[09:02] Marcos`). O escopo é exclusivamente **outbound**: a plataforma envia eventos para os clientes, sem receber webhooks deles (`[09:02–09:03] Marcos, Sofia`).

O ponto de integração crítico é `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que já executa, dentro de uma única transação Prisma (`prisma.$transaction`), a atualização de `orders`, o registro em `order_status_history` e o ajuste de `stock_quantity`. Qualquer solução de notificação precisa se integrar a esse fluxo sem comprometer sua atomicidade nem sua performance.

---

## Proposta Técnica

A solução é composta por cinco decisões arquiteturais centrais, cada uma detalhada em ADR própria (ver [Decisões Relacionadas](#decisões-relacionadas)). Esta seção apresenta a visão geral de como elas se encaixam; o detalhamento de contratos HTTP, payloads e matriz de erros fica no FDD.

**1. Captura atômica do evento (Outbox no MySQL existente).**
Quando `OrderService.changeStatus` transiciona o status de um pedido, uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` insere uma linha na nova tabela `webhook_outbox`, usando o mesmo `Prisma.TransactionClient` (`tx`) já disponível na transação. Se a transação principal falhar, o evento nunca existiu; se ela commitar, o evento está garantidamente persistido. O payload é serializado como **snapshot no momento da inserção**, refletindo o estado exato do pedido na transição, e não é re-renderizado depois (ver ADR-001, ADR-007).

**2. Entrega assíncrona via worker dedicado.**
Um processo Node.js separado (`src/worker.ts`, análogo a `src/server.ts`), com seu próprio `PrismaClient` conectado ao mesmo banco, executa um loop de **polling a cada 2 segundos** sobre a `webhook_outbox`, buscando eventos pendentes em ordem de `created_at`. Rodar como processo separado da API garante que reinícios/deploys da API não interrompam entregas em andamento (ver ADR-002, ADR-003).

**3. Confiabilidade de entrega (retry + DLQ).**
Falhas de entrega (timeout de 10s ou erro HTTP) disparam retry com **backoff exponencial em 5 tentativas** (1m / 5m / 30m / 2h / 12h — janela total de ~15h). Esgotadas as tentativas, o evento é movido para a tabela `webhook_dead_letter`, com payload, motivo da falha e timestamp preservados. Um endpoint administrativo (`POST /admin/webhooks/dead-letter/:id/replay`, protegido por `requireRole('ADMIN')`) permite reprocessamento manual (ver ADR-004).

**4. Autenticidade e integridade (HMAC-SHA256).**
Cada requisição de entrega é assinada com HMAC-SHA256 usando uma **secret exclusiva por endpoint de webhook** (não uma secret global), incluída no header `X-Signature`. Secrets são rotacionáveis, com **grace period de 24h** em que a secret antiga permanece válida em paralelo com a nova (ver ADR-005).

**5. Garantia de entrega at-least-once.**
Cada evento recebe um `event_id` (UUID) gerado na inserção da outbox, propagado em todas as tentativas de entrega via header `X-Event-Id`. O sistema garante que todo evento será entregue **pelo menos uma vez**; a deduplicação de entregas repetidas é responsabilidade do cliente, usando esse identificador estável (ver ADR-006).

**Integração estrutural.** O módulo novo (`src/modules/webhooks/`) segue exatamente a estrutura `controller → service → repository → routes → schemas` já usada em `orders`, `users`, `customers` e `products`, registrado em `buildControllers(prisma)` (`src/app.ts`). Erros de domínio são subclasses de `AppError` (`src/shared/errors/app-error.ts`) com códigos prefixados `WEBHOOK_*`, capturadas pelo `errorMiddleware` existente sem qualquer alteração. Autenticação/autorização usam `authenticate` e `requireRole` já existentes em `src/middlewares/auth.middleware.ts` (ver ADR-008).

Funcionalmente, a proposta também cobre: CRUD de configuração de webhook por cliente (URL, secret, filtro de eventos por status desejado — filtrado já na inserção da outbox), listagem do histórico de entregas por webhook (`GET /webhooks/:id/deliveries`), e o endpoint administrativo de replay de DLQ citado acima. Os contratos completos desses endpoints (payloads de exemplo, status codes) são especificados no FDD.

---

## Alternativas Consideradas

### 1. Chamada HTTP síncrona dentro de `changeStatus`

A abordagem mais direta seria chamar o endpoint do cliente diretamente dentro da transação de mudança de status. Foi descartada porque a transação de `changeStatus` já é pesada (atualiza `orders`, `order_status_history` e estoque), e um endpoint de cliente lento ou indisponível bloquearia a mudança de status de outros pedidos. Além disso, não há uma resposta sensata para "o que fazer se o cliente estiver fora do ar": não é aceitável nem descartar o evento nem fazer rollback da mudança de status (`[09:04] Bruno`, `[09:04] Larissa`).

### 2. Fila dedicada com Redis Streams

Cogitou-se usar Redis Streams como mecanismo de publicação de eventos desacoplado da transação principal. A alternativa foi descartada por exigir infraestrutura nova (Redis Cluster) que o time não opera hoje — considerado overengineering para um time pequeno, dado que o outbox sobre o MySQL já existente resolve o problema sem custo operacional adicional (`[09:07] Larissa`, `[09:07] Diego`).

### 3. Trigger de banco de dados para acionar o worker

Avaliou-se usar triggers MySQL para notificar o worker de forma reativa assim que um evento fosse inserido na outbox, evitando o delay do polling. Foi descartada porque o MySQL não possui um mecanismo nativo equivalente ao `NOTIFY/LISTEN` do PostgreSQL: uma trigger só executa SQL interno, e fazê-la notificar um processo externo exigiria soluções improvisadas (escrita em arquivo, chamada a endpoint), consideradas frágeis demais para o ganho, especialmente porque o polling de 2s já atende ao SLA de "abaixo de 10 segundos" (`[09:09] Bruno`, `[09:09] Diego`).

---

## Questões em Aberto

1. **Rate limiting de envio ao cliente.** Se um cliente tiver muitas mudanças de status em um curto intervalo (ex.: 50 pedidos em 1 minuto), o worker pode disparar um volume alto de chamadas HTTP simultâneas para o mesmo endpoint. A equipe reconheceu o risco mas decidiu não incluí-lo nesta fase — a decisão é observar o comportamento em produção e avaliar a necessidade depois (`[09:38–09:39] Diego`, `[09:39] Larissa`).

2. **Notificação proativa ao cliente sobre falhas recorrentes.** Foi levantada a ideia de avisar o cliente (ex.: por e-mail) quando seu webhook falha repetidamente. A equipe decidiu que isso fica fora do escopo desta fase, a ser reavaliado depois que o impacto da feature for medido (`[09:37] Marcos`, `[09:37] Larissa`).

3. **Ordering de eventos com múltiplos workers.** A ordem de entrega por `order_id` só é garantida enquanto o sistema operar com um único worker processando a outbox por `created_at`. Se for necessário escalar horizontalmente para múltiplos workers no futuro, a garantia de ordering se perde e exigiria particionamento por `order_id` ou lock pessimista — problema explicitamente adiado, sem solução definida nesta proposta (`[09:12–09:13] Diego`, `[09:13] Larissa`).

---

## Impacto e Riscos

- **Novo processo em produção.** O worker (`src/worker.ts`) precisa de deploy, monitoramento e reinício independentes da API — dobra a superfície operacional do sistema (ADR-002).
- **Crescimento não gerenciado da `webhook_outbox`.** Eventos entregues se acumulam na tabela; uma rotina de arquivamento (ex.: após 30 dias) foi mencionada como necessária, mas está fora do escopo desta feature (`[09:08] Diego`).
- **Superfície de segurança nova.** A tabela de configuração de webhooks passa a armazenar secrets por endpoint, exigindo proteção em repouso. Sofia (Segurança) reservou pelo menos dois dias úteis de revisão dedicada ao código de HMAC e geração de secret antes do deploy (`[09:46] Sofia`).
- **Dependência da correta implementação do cliente.** A garantia at-least-once exige que cada cliente implemente deduplicação por `X-Event-Id` do lado dele; falha nessa implementação pode causar processamento duplicado de eventos (ex.: duplo decremento de estoque no sistema do cliente) (ADR-006).
- **Latência mínima intrínseca.** O polling de 2s implica uma latência mínima (não instantânea) entre a mudança de status e o início do envio — aceita pela equipe e pelo PM como compatível com o SLA de "abaixo de 10 segundos" (`[09:10] Larissa`, `[09:10] Marcos`).

---

## Decisões Relacionadas

- [ADR-001 — Uso do Padrão Transactional Outbox no MySQL para Captura Atômica de Eventos de Webhook](adrs/ADR-001-transactional-outbox-pattern-in-mysql.md)
- [ADR-002 — Worker de Webhooks como Processo Node.js Separado](adrs/ADR-002-worker-como-processo-nodejs-separado.md)
- [ADR-003 — Worker de Webhooks com Polling de 2 Segundos na webhook_outbox](adrs/ADR-003-worker-polling-intervalo-dois-segundos.md)
- [ADR-004 — Política de Retry com Backoff Exponencial e Dead Letter Queue para Webhooks](adrs/ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-005 — Autenticação HMAC-SHA256 com Secret por Endpoint e Grace Period de Rotação](adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-006 — Garantia de Entrega At-Least-Once com Deduplicação via X-Event-Id](adrs/ADR-006-garantia-at-least-once-com-x-event-id.md)
- [ADR-007 — Payload da Outbox Armazenado como Snapshot no Momento da Inserção](adrs/ADR-007-payload-snapshot-no-momento-da-insercao.md)
- [ADR-008 — Módulo de Webhook Segue os Padrões Existentes de Controller→Service→Repository e Reutiliza a Infraestrutura Compartilhada](adrs/ADR-008-modulo-webhook-segue-padroes-existentes.md)
