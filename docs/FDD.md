# FDD — Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0
**Data:** 2026-07-06
**Responsável técnico:** Diego (Engenheiro Sênior, time de Plataforma) e Bruno (Engenheiro Pleno, time de Pedidos)
**Documentos relacionados:** [PRD](./PRD.md) · [RFC](./RFC.md) · [ADRs](./adrs/)

---

## 1. Contexto e Motivação Técnica

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) exigem notificação com latência abaixo de 10 segundos quando o status de um pedido muda (`[09:00] Marcos`, `[09:02] Marcos`). Hoje isso é feito por polling manual em `GET /orders`, custoso para os clientes.

O ponto de integração técnico é `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que já executa dentro de `prisma.$transaction` a atualização de `orders`, o registro em `order_status_history` e o débito/reposição de `stockQuantity` via `debitStock`/`replenishStock`. Este FDD detalha como o módulo `webhooks` se integra a esse fluxo sem alterar seu comportamento atual, e como o worker de entrega opera de forma desacoplada.

Este documento assume como já decididas as arquiteturas registradas nas ADRs (outbox transacional, worker separado em polling, retry + DLQ, HMAC-SHA256, at-least-once) — ver [RFC](./RFC.md) e [ADRs](./adrs/). Aqui o foco é o "como construir": fluxos passo a passo, contratos HTTP, matriz de erros e integração real com os arquivos existentes.

**Atores:**
- **Cliente B2B** — consome os webhooks e/ou gerencia sua configuração via API autenticada.
- **Operador/Admin da plataforma** — usuário interno autenticado via JWT (`AuthUser.role`: `ADMIN` | `OPERATOR`), que cadastra webhooks em nome do cliente e (se `ADMIN`) reprocessa a DLQ.
- **API (processo `src/server.ts`)** — grava eventos na outbox dentro da transação de `changeStatus`.
- **Worker (novo processo `src/worker.ts`)** — lê a outbox e entrega os eventos via HTTP.

---

## 2. Objetivos Técnicos

- Garantir atomicidade entre a mudança de status do pedido e o registro do evento de notificação: nunca existe status mudado sem evento registrado, nem evento registrado sem status commitado (invariante do outbox, `[09:40-09:41] Bruno, Diego`).
- Entregar eventos aos clientes com latência de pior caso ≤ 2 segundos de polling + tempo de rede, respeitando o SLA de "abaixo de 10 segundos" (`[09:10] Larissa`, `[09:10] Marcos`).
- Garantir entrega **at-least-once**: todo evento é entregue ao menos uma vez, com `X-Event-Id` estável entre tentativas para deduplicação do lado do cliente (`[09:25] Diego`).
- Esgotar no máximo 5 tentativas de entrega por evento, com janela total de retry de ~15 horas, antes de mover o evento para DLQ (`[09:17] Diego`).
- Garantir autenticidade e integridade de cada entrega via HMAC-SHA256 assinado com secret exclusiva por endpoint (`[09:20-09:21] Sofia`).

---

## 3. Escopo e Exclusões

**Incluído**
- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries`.
- Integração de `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro de `OrderService.changeStatus`.
- Worker (`src/worker.ts` + `WebhookProcessor`) com polling de 2s, retry com backoff exponencial, DLQ.
- CRUD de configuração de webhook (`POST/PATCH/DELETE/GET /webhooks`), filtro de eventos por status.
- Listagem de histórico de entregas (`GET /webhooks/:id/deliveries`).
- Endpoint administrativo de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`), restrito a `ADMIN`.
- Assinatura HMAC-SHA256 dos payloads e rotação de secret com grace period de 24h.

**Excluído (explicitamente descartado ou adiado na reunião)**
- Notificação proativa (e-mail) ao cliente quando webhooks falham repetidamente — adiado para fase futura (`[09:37] Larissa`).
- Rate limiting de envio ao cliente — não implementado nesta fase; equipe decidiu observar em produção antes de agir (`[09:38-09:39] Diego`, `[09:39] Larissa`).
- Dashboard visual para o cliente acompanhar webhooks — fora de escopo, projeto separado do time de frontend (`[09:39-09:40] Larissa`).
- Suporte a webhooks recebidos pela plataforma (inbound) — escopo é somente outbound (`[09:02-09:03] Marcos`, `Sofia`).
- Escalonamento para múltiplos workers paralelos e particionamento por `order_id` — arquitetura atual é single-worker; escala futura é problema em aberto (`[09:12-09:13] Diego`).
- Arquivamento/retenção automática de linhas entregues na `webhook_outbox` (ex.: após 30 dias) — mencionado como necessário, mas fora do escopo desta entrega (`[09:08] Diego`).

---

## 4. Fluxos Detalhados

### 4.1 Criação do evento na outbox (dentro de `changeStatus`)

1. Cliente/operador chama `PATCH /api/v1/orders/:id/status` (rota existente, `order.routes.ts:19-23`), autenticado via `authenticate` (`src/middlewares/auth.middleware.ts`).
2. `OrderController.changeStatus` invoca `OrderService.changeStatus(id, input, userId)` (`order.service.ts:126`).
3. Dentro de `prisma.$transaction(async (tx) => { ... })`:
   a. Busca o `order` atual (`tx.order.findUnique`), valida transição via `canTransition(from, to)` (`order.status.ts`); se inválida, lança `InvalidStatusTransitionError`.
   b. Debita/repõe estoque via `debitStock`/`replenishStock` conforme a transição.
   c. Atualiza `orders.status` e insere em `order_status_history`.
   d. Refaz `tx.order.findUnique` com `include: { items, history, customer }` (linhas 169-176) — o objeto `OrderWithRelations` resultante é a fonte do snapshot.
   e. **[Novo]** Chama `publishWebhookEvent(tx, refreshedOrder, from, to)`, que:
      - Consulta `webhook_endpoints` ativos do `customerId` do pedido cujo filtro de eventos inclui `toStatus`.
      - Se nenhum endpoint estiver interessado, não insere nada na outbox (filtragem ocorre na inserção, `[09:34] Bruno`).
      - Para cada endpoint interessado, gera um `eventId` (UUID v4) e insere uma linha em `webhook_outbox` com `payload` já serializado (snapshot) e `status = PENDING`.
4. Se qualquer etapa falhar, a transação inteira sofre rollback — nenhuma linha de outbox é criada sem o commit da mudança de status, e vice-versa (ADR-001).
5. A resposta HTTP de `PATCH /orders/:id/status` retorna o pedido atualizado, sem esperar qualquer entrega de webhook (a entrega é assíncrona).

### 4.2 Processamento pelo worker

1. `src/worker.ts` inicia, instancia `PrismaClient` próprio via `createPrismaClient()` (`src/config/database.ts:4-8`) e entra em loop de polling a cada 2 segundos.
2. A cada ciclo, `WebhookProcessor` executa uma query em `webhook_outbox` filtrando `status = PENDING AND nextRetryAt <= NOW()`, ordenada por `createdAt ASC`, em batch limitado.
3. Para cada evento pendente:
   a. Busca o `webhook_endpoints` associado (URL, secret ativa e, se em grace period, secret anterior).
   b. Assina o `payload` (já materializado na outbox) com HMAC-SHA256 usando a secret ativa; monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`).
   c. Executa a chamada HTTP `POST` para a URL do endpoint com timeout de 10 segundos.
   d. Registra o resultado (sucesso/falha, status HTTP, tempo de resposta) em `webhook_deliveries`.
4. Em caso de sucesso (2xx): marca a linha da outbox como `DELIVERED` (ou remove, conforme retenção futura).
5. Em caso de falha (timeout, erro de rede, status ≥ 400): segue para o fluxo de retry (4.3).

### 4.3 Retry com backoff exponencial

1. Ao falhar, o worker incrementa `attemptCount` no registro da outbox e calcula `nextRetryAt = NOW() + backoff(attemptCount)`, com a progressão:

   | Tentativa | Intervalo até a próxima |
   | --- | --- |
   | 1 → 2 | 1 minuto |
   | 2 → 3 | 5 minutos |
   | 3 → 4 | 30 minutos |
   | 4 → 5 | 2 horas |
   | 5 → esgotado | 12 horas |

2. O evento permanece com `status = PENDING` (ou `RETRYING`) na outbox até `attemptCount` atingir 5 tentativas falhas.
3. O ciclo de polling (4.2) ignora o evento até `nextRetryAt` ser alcançado (índice composto `(status, nextRetryAt)`).

### 4.4 Movimentação para Dead Letter Queue (DLQ)

1. Após a 5ª tentativa falha, o worker:
   a. Insere uma linha em `webhook_dead_letter` com `payload`, `reason` (motivo/última resposta de erro) e `failedAt`.
   b. Remove (ou marca como `FAILED`, removendo do fluxo de polling) o registro correspondente em `webhook_outbox`.
2. Um operador com role `ADMIN` pode disparar `POST /admin/webhooks/dead-letter/:id/replay` (protegido por `requireRole('ADMIN')`, `src/middlewares/auth.middleware.ts:49-61`).
3. O replay reinsere o evento em `webhook_outbox` com `status = PENDING`, `attemptCount = 0`, e o worker o processa no próximo ciclo. A ação é registrada em log estruturado (Pino) com o `userId` do operador, para auditoria (`[09:36] Sofia`).

---

## 5. Contratos Públicos

Todas as rotas seguem o padrão de composição existente: `authenticate` (JWT) é aplicado a todas; `requireRole('ADMIN')` é aplicado apenas ao replay de DLQ, replicando o padrão de `order.routes.ts`.

### 5.1 `POST /api/v1/webhooks` — Cadastrar webhook

- **Autenticação:** `authenticate` (qualquer role autenticada, `[09:36-09:37] Sofia`).
- **Request:**
```json
{
  "customerId": "b7e1c9b0-6b1a-4e8e-9c3a-2f2e6b1a1234",
  "url": "https://integracoes.atlascomercial.com.br/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```
- **Response `201 Created`:**
```json
{
  "id": "d290f1ee-6c54-4b01-90e6-d701748f0851",
  "customerId": "b7e1c9b0-6b1a-4e8e-9c3a-2f2e6b1a1234",
  "url": "https://integracoes.atlascomercial.com.br/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_5f8a1c2e9b3d4f6a7c8e9b0d1f2a3c4e",
  "active": true,
  "createdAt": "2026-07-06T12:00:00.000Z"
}
```
- **Semântica:** a `secret` é gerada pelo servidor e retornada **apenas nesta resposta de criação** (`[09:31] Marcos`). `url` deve ser `https://`; caso contrário, `422 WEBHOOK_INVALID_URL`.
- **Status codes:** `201` sucesso · `400 VALIDATION_ERROR` (schema Zod) · `401 UNAUTHORIZED` · `404 WEBHOOK_CUSTOMER_NOT_FOUND` (customerId inexistente) · `422 WEBHOOK_INVALID_URL`.

### 5.2 `PATCH /api/v1/webhooks/:id` — Editar webhook

- **Autenticação:** `authenticate`.
- **Request:**
```json
{
  "url": "https://integracoes.atlascomercial.com.br/webhooks/v2/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```
- **Response `200 OK`:**
```json
{
  "id": "d290f1ee-6c54-4b01-90e6-d701748f0851",
  "customerId": "b7e1c9b0-6b1a-4e8e-9c3a-2f2e6b1a1234",
  "url": "https://integracoes.atlascomercial.com.br/webhooks/v2/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-07-06T14:30:00.000Z"
}
```
- **Status codes:** `200` sucesso · `400 VALIDATION_ERROR` · `401 UNAUTHORIZED` · `404 WEBHOOK_NOT_FOUND` · `422 WEBHOOK_INVALID_URL`.

### 5.3 `DELETE /api/v1/webhooks/:id` — Remover webhook

- **Autenticação:** `authenticate`.
- **Response:** `204 No Content` (sem corpo), seguindo o padrão de `OrderController.delete` (`order.controller.ts:48-55`).
- **Status codes:** `204` sucesso · `401 UNAUTHORIZED` · `404 WEBHOOK_NOT_FOUND`.

### 5.4 `GET /api/v1/webhooks?customerId=...` — Listar webhooks de um customer

- **Autenticação:** `authenticate`.
- **Response `200 OK`:**
```json
{
  "items": [
    {
      "id": "d290f1ee-6c54-4b01-90e6-d701748f0851",
      "customerId": "b7e1c9b0-6b1a-4e8e-9c3a-2f2e6b1a1234",
      "url": "https://integracoes.atlascomercial.com.br/webhooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-06T12:00:00.000Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```
- **Semântica:** segue o padrão de resposta paginada `paginated(...)` já usado em `OrderService.list` (`src/shared/http/response.ts`).
- **Status codes:** `200` sucesso · `401 UNAUTHORIZED`.

### 5.5 `GET /api/v1/webhooks/:id/deliveries` — Histórico de entregas

- **Autenticação:** `authenticate`.
- **Response `200 OK`:**
```json
{
  "items": [
    {
      "id": "6f9619ff-8b86-d011-b42d-00cf4fc964ff",
      "eventId": "6d0f1a2b-3c4d-4e5f-8a9b-0c1d2e3f4a5b",
      "eventType": "order.status_changed",
      "attempt": 1,
      "success": true,
      "responseStatus": 200,
      "responseTimeMs": 184,
      "deliveredAt": "2026-07-06T12:00:02.140Z"
    },
    {
      "id": "7a8b9c0d-1e2f-4a3b-9c4d-5e6f7a8b9c0d",
      "eventId": "9c0d1e2f-3a4b-4c5d-9e6f-7a8b9c0d1e2f",
      "eventType": "order.status_changed",
      "attempt": 3,
      "success": false,
      "responseStatus": 503,
      "responseTimeMs": 10000,
      "deliveredAt": "2026-07-06T13:35:00.000Z"
    }
  ],
  "page": 1,
  "pageSize": 100,
  "total": 2
}
```
- **Semântica:** retorna as últimas entregas (sucesso/falha) do webhook, incluindo `responseStatus` e `responseTimeMs` (`[09:34] Marcos`).
- **Status codes:** `200` sucesso · `401 UNAUTHORIZED` · `404 WEBHOOK_NOT_FOUND`.

### 5.6 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — Replay manual de DLQ

- **Autenticação:** `authenticate` + `requireRole('ADMIN')` (`[09:36] Larissa`, `src/middlewares/auth.middleware.ts:49-61`).
- **Request:** sem corpo.
- **Response `200 OK`:**
```json
{
  "id": "9c0d1e2f-3a4b-4c5d-9e6f-7a8b9c0d1e2f",
  "status": "PENDING",
  "attemptCount": 0,
  "requeuedAt": "2026-07-06T15:00:00.000Z",
  "requeuedBy": "3f2504e0-4f89-11d3-9a0c-0305e82c3301"
}
```
- **Semântica:** reinsere o evento na `webhook_outbox` com `status = PENDING`; ação registrada em log Pino estruturado com `requeuedBy` (id do operador ADMIN) para auditoria.
- **Status codes:** `200` sucesso · `401 UNAUTHORIZED` · `403 FORBIDDEN` (role ≠ ADMIN) · `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 5.7 Requisição de entrega do webhook (worker → cliente)

- **Método:** `POST` para a `url` cadastrada no `webhook_endpoints`.
- **Headers:**

| Header | Descrição |
| --- | --- |
| `X-Event-Id` | UUID estável do evento, igual em todas as tentativas (dedup do lado do cliente) (`[09:25] Diego`) |
| `X-Signature` | HMAC-SHA256 do corpo, hex-encoded, calculado com a secret do endpoint (`[09:20] Sofia`) |
| `X-Timestamp` | Timestamp ISO 8601 do envio, para detecção de replay attack pelo cliente (`[09:44] Diego`) |
| `X-Webhook-Id` | Id do cadastro de webhook que originou o envio (`[09:44] Sofia`) |
| `Content-Type` | `application/json` |

- **Body de exemplo:**
```json
{
  "event_id": "6d0f1a2b-3c4d-4e5f-8a9b-0c1d2e3f4a5b",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-06T12:00:00.000Z",
  "order_id": "e4b1a4a0-1c2d-4e3f-9a5b-6c7d8e9f0a1b",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "b7e1c9b0-6b1a-4e8e-9c3a-2f2e6b1a1234",
  "total_cents": 15990
}
```
- **Semântica:** payload não inclui `items`; se o cliente precisar de detalhes, deve consultar `GET /orders/:id` (`[09:43] Diego`). Timeout de resposta esperado do cliente: 10 segundos.
- **Limites:** tamanho máximo do payload 64 KB — se excedido, o evento não é enviado e é registrado como erro (`[09:23-09:24] Sofia, Diego`).

---

## 6. Matriz de Erros

| Código | Status HTTP | Condição | Tratamento |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `PATCH`/`DELETE`/`GET deliveries` com `id` inexistente | Retornar 404, sem side-effects |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` informado não existe | Retornar 404 antes de criar o cadastro |
| `WEBHOOK_INVALID_URL` | 422 | `url` cadastrada não é `https://` | Rejeitado na validação Zod/schema, endpoint não é criado (`[09:23] Sofia`) |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Operação exige secret ativa e nenhuma está disponível (ex.: grace period expirado sem rotação concluída) | Retornar erro; endpoint não pode receber entregas até nova secret ser configurada |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload serializado do evento excede 64 KB | Evento não é inserido/entregue; registrado como falha, não é retentado (`[09:23-09:24] Sofia, Diego`) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de DLQ com `id` inexistente | Retornar 404 |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno ao worker) | Chamada HTTP ao cliente não responde em 10s | Tratado como falha de entrega; segue fluxo de retry (`[09:42] Diego`) |
| `VALIDATION_ERROR` | 400 | Payload de request não passa nos schemas Zod | Reaproveita `ValidationError`/`ZodError` já tratado pelo `errorMiddleware` existente (`src/middlewares/error.middleware.ts:26-35`), sem alteração |
| `UNAUTHORIZED` | 401 | Ausência/invalidez de JWT | Reaproveita `UnauthorizedError` existente (`src/shared/errors/http-errors.ts:15-19`) |
| `FORBIDDEN` | 403 | Role ≠ `ADMIN` no replay de DLQ | Reaproveita `ForbiddenError` existente (`src/shared/errors/http-errors.ts:21-25`) via `requireRole('ADMIN')` |

Todas as classes de erro específicas do domínio (`WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookSecretRequiredError`, `WebhookPayloadTooLargeError`, `WebhookDeadLetterNotFoundError`) estendem `AppError` (`src/shared/errors/app-error.ts`), seguindo exatamente o modelo de `InvalidStatusTransitionError`/`InsufficientStockError` (`src/shared/errors/http-errors.ts:45-63`). O `errorMiddleware` (`src/middlewares/error.middleware.ts:14-24`) já captura genericamente qualquer `instanceof AppError` — **nenhuma alteração é necessária nesse arquivo**.

---

## 7. Estratégias de Resiliência

- **Timeout por chamada HTTP de entrega:** 10 segundos (`[09:42] Diego`). Excedido, é tratado como falha.
- **Retry com backoff exponencial:** 5 tentativas, intervalos 1m/5m/30m/2h/12h (ver seção 4.3, ADR-004).
- **Dead Letter Queue:** isola definitivamente eventos que esgotaram as tentativas, preservando payload e motivo para diagnóstico e replay manual (ADR-004).
- **Fallback:** não há fallback automático de canal (ex.: e-mail) — deliberadamente fora de escopo (`[09:37] Larissa`). A única forma de recuperação de um evento em DLQ é o replay manual via endpoint admin.
- **Isolamento de falha entre processos:** worker roda em processo separado (`src/worker.ts`); uma falha ou reinício da API não interrompe entregas em andamento, e vice-versa (ADR-002).
- **Rejeição por tamanho:** payload > 64 KB é recusado no momento da inserção na outbox, não é retentado (evento malformado, não transiente).

---

## 8. Observabilidade

**Métricas** (via logging estruturado Pino, sem infraestrutura de métricas nova introduzida nesta fase):
- Contagem de eventos inseridos na `webhook_outbox` por `status` e por `customerId`.
- Contagem e latência (`responseTimeMs`) das entregas em `webhook_deliveries`, segmentada por sucesso/falha e por tentativa (`attempt`).
- Contagem de eventos movidos para `webhook_dead_letter` por dia.
- Tempo entre a inserção do evento na outbox e a primeira tentativa de entrega (proxy da latência real de ponta a ponta).

**Logs** (Pino, reaproveitando `src/shared/logger/index.ts`, sem alterações no logger em si):
- API: log estruturado na inserção de evento na outbox (`orderId`, `eventId`, `toStatus`, `customerId`), no padrão de campos já usado pelo `requestLogger` existente.
- Worker: log a cada ciclo de polling com contagem de eventos processados; log de erro em cada falha de entrega (`eventId`, `webhookId`, `attempt`, `responseStatus`/erro de rede) — mesmo padrão de redação de campos sensíveis já configurado em `redactPaths` (`src/shared/logger/index.ts:4-11`), que deve ser estendido para nunca logar o campo `secret`.
- Replay de DLQ: log obrigatório de auditoria com `userId` do operador ADMIN, `deadLetterId` e timestamp (`[09:36] Sofia`).

**Tracing:**
- Não há infraestrutura de tracing distribuído no projeto atual; não introduzida nesta feature. Rastreabilidade de uma entrega específica é feita por correlação manual via `eventId` entre `webhook_outbox`, `webhook_deliveries` e (se aplicável) `webhook_dead_letter`.

---

## 9. Integração com o Sistema Existente

- **`src/modules/orders/order.service.ts`** (método `changeStatus`, linhas 126-179): ponto de integração crítico. Após o `tx.order.findUnique` que recarrega o pedido com relações (linhas 169-176), uma chamada a `publishWebhookEvent(tx, refreshedOrder, from, to)` é adicionada dentro do mesmo `prisma.$transaction`, garantindo atomicidade entre a mudança de status e o registro do evento na outbox. Nenhuma outra lógica do método é alterada.
- **`src/shared/errors/app-error.ts`** e **`src/shared/errors/http-errors.ts`**: as novas classes de erro do módulo (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendem `AppError`, seguindo o mesmo padrão de `InvalidStatusTransitionError` e `InsufficientStockError` já implementadas nesses arquivos — nenhuma modificação nos arquivos existentes, apenas novas classes em `src/modules/webhooks/`.
- **`src/middlewares/error.middleware.ts`**: captura automaticamente qualquer `instanceof AppError`, `ZodError` e erros do Prisma (`P2002`, `P2025`). Como as novas classes de erro seguem o mesmo contrato de `AppError`, o middleware funciona sem alteração para o módulo de webhooks.
- **`src/middlewares/auth.middleware.ts`**: `authenticate` protege todas as rotas do módulo de webhooks; `requireRole('ADMIN')` (linhas 49-61) protege especificamente `POST /admin/webhooks/dead-letter/:id/replay`, reaproveitado sem qualquer mudança de assinatura.
- **`src/middlewares/validate.middleware.ts`**: o `validate({ body, params, query })` existente é aplicado aos novos schemas Zod do módulo de webhooks (`webhook.schemas.ts`), seguindo o padrão de `order.routes.ts:16-24`.
- **`src/app.ts`** (`buildControllers(prisma)`, linhas 26-53): recebe a instanciação de `WebhookRepository`, `WebhookService` e `WebhookController`, adicionados ao objeto `Controllers` retornado, sem alterar a mecânica de composição manual existente.
- **`src/routes/index.ts`** (`buildApiRouter`, linhas 21-31): recebe `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks))`, seguindo o padrão de registro de sub-routers já usado para `orders`, `customers`, `products`.
- **`src/config/database.ts`** (`createPrismaClient()`, linhas 4-8): reutilizada sem alteração para instanciar o `PrismaClient` independente do worker em `src/worker.ts` (novo arquivo, mesma `DATABASE_URL`).
- **`src/server.ts`**: serve de modelo estrutural (`bootstrap()`, handlers `SIGINT`/`SIGTERM`, `prisma.$disconnect()`) para o novo `src/worker.ts`, que replica o mesmo padrão de ciclo de vida de processo.
- **`src/shared/logger/index.ts`**: o logger Pino é importado sem alteração pelo worker e pela API; o array `redactPaths` (linhas 4-11) deve ser estendido para incluir o campo `secret` dos endpoints de webhook.
- **`prisma/schema.prisma`**: recebe os novos modelos `WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter` e `WebhookDelivery`, seguindo as convenções já usadas em todo o schema — `id String @id @default(uuid()) @db.Char(36)`, `createdAt`/`updatedAt`, `@@index(...)` nos campos de filtro (`status`, `createdAt`, `nextRetryAt`) e `@@map("snake_case")` (ex.: `@@map("webhook_outbox")`), no mesmo padrão de `Order`, `OrderStatusHistory` (linhas 74-131).

---

## 10. Critérios de Aceite Técnicos

- Ao mudar o status de um pedido via `PATCH /orders/:id/status`, uma linha correspondente aparece em `webhook_outbox` (para cada webhook do customer inscrito naquele status) se e somente se a transação de `changeStatus` foi commitada — nunca em rollback.
- Um evento de teste é entregue ao endpoint de destino em, no máximo, 2 ciclos de polling (~4 segundos) após a inserção na outbox, quando o endpoint está disponível.
- Uma falha simulada de entrega gera até 5 tentativas nos intervalos definidos (1m/5m/30m/2h/12h) antes de mover o evento para `webhook_dead_letter`.
- O header `X-Signature` de uma entrega, recalculado pelo receptor com a secret correta, corresponde à assinatura HMAC-SHA256 do corpo exato recebido.
- Durante o grace period de 24h após rotação de secret, entregas assinadas tanto com a secret antiga quanto com a nova são aceitas pelo worker como válidas.
- Todo evento entregue (inclusive reentregas) carrega o mesmo `X-Event-Id` em todas as tentativas.
- `POST /admin/webhooks/dead-letter/:id/replay` sem role `ADMIN` retorna `403 FORBIDDEN`; com role `ADMIN`, reinsere o evento na outbox com `status = PENDING`.
- Payload de evento acima de 64 KB não é inserido na outbox e gera erro `WEBHOOK_PAYLOAD_TOO_LARGE`.
- Endpoint cadastrado com `url` não-`https` é rejeitado com `422 WEBHOOK_INVALID_URL`.

---

## 11. Riscos e Mitigação

**Risco 1 — Volume alto de mudanças de status em curto intervalo gera pico de chamadas HTTP a um mesmo cliente**
- Probabilidade: média
- Impacto: possível sobrecarga do endpoint do cliente, aumento de falhas e retries
- Mitigação:
  - Nenhuma ação de rate limiting implementada nesta fase — decisão consciente da equipe (`[09:38-09:39] Diego, Larissa`)
  - Observar métricas de volume por customer após o lançamento
- Plano de contingência: se o problema se materializar, avaliar rate limiting de saída por endpoint em fase futura (fora do escopo atual)

**Risco 2 — Crescimento não controlado da tabela `webhook_outbox`**
- Probabilidade: média
- Impacto: degradação de performance do polling do worker ao longo do tempo
- Mitigação:
  - Índices compostos `(status, nextRetryAt)` e `(createdAt)` desde o início (ADR-003)
- Plano de contingência: implementar rotina de arquivamento/expurgo de linhas `DELIVERED` após 30 dias em iteração futura (mencionado, não implementado agora, `[09:08] Diego`)

**Risco 3 — Vazamento de secret de um cliente**
- Probabilidade: baixa (já ocorreu um incidente similar relatado pela equipe, `[09:22] Diego`)
- Impacto: possibilidade de forjar entregas para o endpoint daquele cliente até a rotação
- Mitigação:
  - Secret exclusiva por endpoint (blast radius limitado a um cliente, ADR-005)
  - Suporte a rotação self-service com grace period de 24h, sem downtime de verificação
  - Revisão de segurança dedicada da Sofia antes do deploy (`[09:46] Sofia`)
- Plano de contingência: cliente aciona rotação de secret via API assim que suspeitar de vazamento

**Risco 4 — Cliente não implementa deduplicação por `X-Event-Id` corretamente**
- Probabilidade: média
- Impacto: processamento duplicado de eventos no sistema do cliente (ex.: duplo decremento de estoque do lado dele)
- Mitigação:
  - Documentação clara e destacada no portal de desenvolvedor sobre a semântica at-least-once (responsabilidade de Marcos, `[09:26] Marcos`)
  - Alinhamento com padrão de mercado (Stripe, GitHub) já conhecido por integradores B2B (`[09:25] Diego`)
- Plano de contingência: nenhum mecanismo automático da plataforma — mitigação depende de comunicação e suporte ao cliente durante onboarding
