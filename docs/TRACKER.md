# Tracker de Rastreabilidade

Este documento mapeia cada item relevante registrado em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e nas ADRs (`docs/adrs/ADR-*.md`) à sua origem: um timestamp específico em `TRANSCRICAO.md` ou um caminho de arquivo real em `src/`/`prisma/`. A tabela está dividida por documento de origem para facilitar a leitura, mas segue exatamente o mesmo esquema de colunas em toda a extensão.

Convenção de IDs: `[DOC]-[TIPO]-[NN]`, por exemplo `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-004-DEC`.

---

## docs/PRD.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Restrição | Escopo é exclusivamente outbound (plataforma → cliente), sem receber webhooks | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | Risco comercial: Atlas Comercial pode migrar para concorrente se prazo não for cumprido | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | Objetivo com métrica: latência de entrega ≤ 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Decisão | Objetivo: 100% das mudanças de status geram evento (atomicidade) | TRANSCRICAO | [09:04] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Decisão | Objetivo: janela de retry ≥ 15h antes de falha definitiva | TRANSCRICAO | [09:17] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Notificação automática de mudança de status via webhook | TRANSCRICAO | [09:03] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (URL, secret gerada, status desejados) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição de webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listagem de webhooks por cliente (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status na inscrição do webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico de entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Retry automático com backoff exponencial e DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Replay manual de eventos em DLQ (role ADMIN) | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado para não afetar disponibilidade da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Toda entrega assinada com HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | URL de destino do webhook deve ser HTTPS obrigatoriamente | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Replay de DLQ exige role ADMIN e log de auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Captura do evento atômica com a mudança de status | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Garantia de entrega at-least-once com X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Payload do evento limitado a 64 KB | TRANSCRICAO | [09:23] Sofia |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Fora de escopo: notificação proativa (e-mail) em falhas recorrentes | TRANSCRICAO | [09:37] Marcos |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de envio ao cliente | TRANSCRICAO | [09:38] Diego |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual para o cliente | TRANSCRICAO | [09:39] Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição | Fora de escopo: recebimento de webhooks de terceiros (inbound) | TRANSCRICAO | [09:02] Marcos |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição | Fora de escopo: múltiplos workers paralelos com ordering global | TRANSCRICAO | [09:12] Diego |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox transacional no MySQL existente, sem fila externa | TRANSCRICAO | [09:06] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker em processo separado com polling de 2s | TRANSCRICAO | [09:09] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão | Retry com 5 tentativas, backoff exponencial, DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-DEC-04 | docs/PRD.md | Decisão | HMAC-SHA256 com secret por endpoint e grace period de 24h | TRANSCRICAO | [09:20] Sofia |
| PRD-DEC-05 | docs/PRD.md | Trade-off | At-least-once com X-Event-Id em vez de exactly-once | TRANSCRICAO | [09:24] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Documentação no portal de desenvolvedor sobre dedup e HMAC | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança dedicada da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Confirmação de prazo com clientes B2B após planejamento de sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-01 | docs/PRD.md | Restrição | Risco: alto volume de status pode gerar pico de chamadas a um cliente | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-02 | docs/PRD.md | Restrição | Risco: vazamento de secret de cliente (incidente real já ocorrido) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Restrição | Risco: crescimento não controlado da tabela de outbox | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-04 | docs/PRD.md | Restrição | Risco: cliente não implementa deduplicação corretamente | TRANSCRICAO | [09:25] Diego |
| PRD-AC-01 | docs/PRD.md | Requisito Funcional | Critério de aceite: entrega em até 10s com endpoint disponível | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | docs/PRD.md | Requisito Funcional | Critério de aceite: retry segue 1m/5m/30m/2h/12h antes de DLQ | TRANSCRICAO | [09:17] Diego |
| PRD-AC-03 | docs/PRD.md | Requisito Funcional | Critério de aceite: cadastro com URL não-HTTPS é rejeitado | TRANSCRICAO | [09:23] Sofia |

## docs/RFC.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | docs/RFC.md | Restrição | Requisito de latência aceito como "tempo real": abaixo de 10s | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-02 | docs/RFC.md | Restrição | Ponto de integração crítico é OrderService.changeStatus | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | publishWebhookEvent(tx, order, fromStatus, toStatus) dentro da transação | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker Node.js separado (src/worker.ts) em polling de 2s | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Módulo webhooks segue padrão controller/service/repository | CODIGO | src/app.ts |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Alternativa descartada: chamada HTTP síncrona dentro de changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Alternativa descartada: Redis Streams (overengineering para time pequeno) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Alternativa descartada: trigger de banco para acionar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Questão em aberto: rate limiting de envio ao cliente | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Questão em aberto: notificação proativa sobre falhas recorrentes | TRANSCRICAO | [09:37] Marcos |
| RFC-OPEN-03 | docs/RFC.md | Restrição | Questão em aberto: ordering com múltiplos workers no futuro | TRANSCRICAO | [09:12] Diego |
| RFC-IMPACT-01 | docs/RFC.md | Restrição | Impacto: novo processo (worker) a operar e monitorar | TRANSCRICAO | [09:11] Diego |
| RFC-IMPACT-02 | docs/RFC.md | Restrição | Impacto: crescimento da webhook_outbox sem arquivamento automático | TRANSCRICAO | [09:08] Diego |
| RFC-IMPACT-03 | docs/RFC.md | Restrição | Impacto: nova superfície de segurança (secrets armazenadas) | TRANSCRICAO | [09:46] Sofia |
| RFC-IMPACT-04 | docs/RFC.md | Restrição | Impacto: dependência da correta dedup pelo cliente | TRANSCRICAO | [09:25] Diego |
| RFC-IMPACT-05 | docs/RFC.md | Restrição | Impacto: latência mínima intrínseca de 2s do polling | TRANSCRICAO | [09:10] Larissa |

## docs/FDD.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLOW-01 | docs/FDD.md | Requisito Funcional | Fluxo: criação do evento na outbox dentro de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLOW-02 | docs/FDD.md | Requisito Funcional | Fluxo: processamento pelo worker (polling, assinatura, dispatch) | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-03 | docs/FDD.md | Requisito Funcional | Fluxo: retry com backoff exponencial (tabela de intervalos) | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-04 | docs/FDD.md | Requisito Funcional | Fluxo: movimentação para DLQ e replay administrativo | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /api/v1/webhooks — cadastrar webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | PATCH /api/v1/webhooks/:id — editar webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | DELETE /api/v1/webhooks/:id — remover webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | GET /api/v1/webhooks — listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | GET /api/v1/webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | POST /api/v1/admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | Requisição de entrega worker→cliente: headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| FDD-ERRO-01 | docs/FDD.md | Restrição | WEBHOOK_NOT_FOUND (404) | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-02 | docs/FDD.md | Restrição | WEBHOOK_INVALID_URL (422) — URL não HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Restrição | WEBHOOK_SECRET_REQUIRED (422) | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-04 | docs/FDD.md | Restrição | WEBHOOK_PAYLOAD_TOO_LARGE (422) — limite 64 KB | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-05 | docs/FDD.md | Restrição | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-06 | docs/FDD.md | Restrição | WEBHOOK_DELIVERY_TIMEOUT — timeout de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-07 | docs/FDD.md | Restrição | VALIDATION_ERROR reaproveitado do errorMiddleware existente | CODIGO | src/middlewares/error.middleware.ts |
| FDD-ERRO-08 | docs/FDD.md | Restrição | UNAUTHORIZED reaproveitado de UnauthorizedError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-09 | docs/FDD.md | Restrição | FORBIDDEN reaproveitado de ForbiddenError via requireRole | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por chamada de entrega | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Requisito Não Funcional | Retry com backoff exponencial, 5 tentativas | TRANSCRICAO | [09:15] Diego |
| FDD-RESIL-03 | docs/FDD.md | Requisito Não Funcional | Dead Letter Queue isola falhas definitivas | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-04 | docs/FDD.md | Restrição | Sem fallback automático de canal (e-mail fora de escopo) | TRANSCRICAO | [09:37] Larissa |
| FDD-RESIL-05 | docs/FDD.md | Requisito Não Funcional | Isolamento de falha entre processos API/worker | TRANSCRICAO | [09:11] Diego |
| FDD-RESIL-06 | docs/FDD.md | Restrição | Payload > 64 KB rejeitado sem retry | TRANSCRICAO | [09:23] Sofia |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Métricas via logging estruturado de eventos e entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Logs Pino reaproveitados, redactPaths estendido para secret | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Restrição | Log de auditoria obrigatório no replay de DLQ | TRANSCRICAO | [09:36] Sofia |
| FDD-INTEG-01 | docs/FDD.md | Restrição | Integração: publishWebhookEvent dentro de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição | Integração: novas classes de erro estendem AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | Integração: erros seguem modelo de InvalidStatusTransitionError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição | Integração: errorMiddleware captura AppError sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição | Integração: authenticate/requireRole reaproveitados | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição | Integração: validate aplicado aos novos schemas Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INTEG-07 | docs/FDD.md | Restrição | Integração: WebhookController registrado em buildControllers | CODIGO | src/app.ts |
| FDD-INTEG-08 | docs/FDD.md | Restrição | Integração: sub-router registrado em buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INTEG-09 | docs/FDD.md | Restrição | Integração: createPrismaClient() reutilizado pelo worker | CODIGO | src/config/database.ts |
| FDD-INTEG-10 | docs/FDD.md | Restrição | Integração: src/worker.ts espelha padrão de src/server.ts | CODIGO | src/server.ts |
| FDD-INTEG-11 | docs/FDD.md | Restrição | Integração: logger Pino importado sem alteração pelo worker | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-12 | docs/FDD.md | Restrição | Integração: novos modelos seguem convenções de schema.prisma | CODIGO | prisma/schema.prisma |
| FDD-AC-01 | docs/FDD.md | Requisito Funcional | Critério: evento gerado apenas se transação de changeStatus for commitada | TRANSCRICAO | [09:04] Bruno |
| FDD-AC-02 | docs/FDD.md | Requisito Funcional | Critério: entrega em até 2 ciclos de polling (~4s) | TRANSCRICAO | [09:09] Diego |
| FDD-AC-03 | docs/FDD.md | Requisito Funcional | Critério: grace period aceita assinatura com secret antiga e nova | TRANSCRICAO | [09:21] Sofia |
| FDD-RISK-01 | docs/FDD.md | Restrição | Risco: volume alto gera pico de chamadas a um cliente | TRANSCRICAO | [09:38] Diego |
| FDD-RISK-02 | docs/FDD.md | Restrição | Risco: crescimento não controlado da webhook_outbox | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-03 | docs/FDD.md | Restrição | Risco: vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-04 | docs/FDD.md | Restrição | Risco: cliente não implementa dedup por X-Event-Id | TRANSCRICAO | [09:25] Diego |

## docs/adrs/ADR-001-transactional-outbox-pattern-in-mysql.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001-DEC | docs/adrs/ADR-001-transactional-outbox-pattern-in-mysql.md | Decisão | Outbox transacional no MySQL, publishWebhookEvent dentro da transação | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-transactional-outbox-pattern-in-mysql.md | Trade-off | Alternativa descartada: chamada HTTP síncrona em changeStatus | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-transactional-outbox-pattern-in-mysql.md | Trade-off | Alternativa descartada: Redis Streams / Redis Cluster | TRANSCRICAO | [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-transactional-outbox-pattern-in-mysql.md | Trade-off | Alternativa descartada: trigger de banco MySQL | TRANSCRICAO | [09:09] Diego |
| ADR-001-COD-01 | docs/adrs/ADR-001-transactional-outbox-pattern-in-mysql.md | Restrição | Transação já existente em changeStatus (tx.$transaction) | CODIGO | src/modules/orders/order.service.ts |

## docs/adrs/ADR-002-worker-como-processo-nodejs-separado.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-002-DEC | docs/adrs/ADR-002-worker-como-processo-nodejs-separado.md | Decisão | Worker como processo Node.js independente (src/worker.ts) | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-como-processo-nodejs-separado.md | Trade-off | Alternativa descartada: setInterval dentro do processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-como-processo-nodejs-separado.md | Trade-off | Alternativa não aprofundada: microsserviço em linguagem/container separado | TRANSCRICAO | [09:07] Diego |
| ADR-002-COD-01 | docs/adrs/ADR-002-worker-como-processo-nodejs-separado.md | Restrição | Padrão de bootstrap/SIGINT/SIGTERM a ser espelhado | CODIGO | src/server.ts |
| ADR-002-COD-02 | docs/adrs/ADR-002-worker-como-processo-nodejs-separado.md | Restrição | createPrismaClient() suporta múltiplas instâncias por processo | CODIGO | src/config/database.ts |

## docs/adrs/ADR-003-worker-polling-intervalo-dois-segundos.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-003-DEC | docs/adrs/ADR-003-worker-polling-intervalo-dois-segundos.md | Decisão | Polling a cada 2 segundos na webhook_outbox | TRANSCRICAO | [09:09] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-worker-polling-intervalo-dois-segundos.md | Trade-off | Alternativa descartada: MySQL triggers com notificação externa | TRANSCRICAO | [09:09] Bruno |
| ADR-003-ALT-02 | docs/adrs/ADR-003-worker-polling-intervalo-dois-segundos.md | Trade-off | Alternativa descartada: fila baseada em Redis Streams | TRANSCRICAO | [09:07] Larissa |
| ADR-003-ALT-03 | docs/adrs/ADR-003-worker-polling-intervalo-dois-segundos.md | Trade-off | Alternativa não debatida: intervalo mais curto (ex.: 500ms) | TRANSCRICAO | [09:09] Diego |
| ADR-003-COD-01 | docs/adrs/ADR-003-worker-polling-intervalo-dois-segundos.md | Restrição | MySQL como provider sem NOTIFY/LISTEN | CODIGO | prisma/schema.prisma |

## docs/adrs/ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-004-DEC | docs/adrs/ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ separada | TRANSCRICAO | [09:17] Diego |
| ADR-004-ALT-01 | docs/adrs/ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Alternativa descartada: 3 tentativas mais agressivas | TRANSCRICAO | [09:16] Bruno |
| ADR-004-ALT-02 | docs/adrs/ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Alternativa descartada: retry indefinido | TRANSCRICAO | [09:16] Larissa |
| ADR-004-COD-01 | docs/adrs/ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md | Restrição | requireRole reaproveitado para o replay de DLQ | CODIGO | src/middlewares/auth.middleware.ts |

## docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-005-DEC | docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e grace period de 24h | TRANSCRICAO | [09:20] Sofia |
| ADR-005-ALT-01 | docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Alternativa descartada: secret global única da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-005-ALT-02 | docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Alternativa descartada: API Key sem assinatura de payload | TRANSCRICAO | [09:20] Sofia |
| ADR-005-ALT-03 | docs/adrs/ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Alternativa não debatida: allowlist de IPs sem assinatura | TRANSCRICAO | [09:19] Sofia |

## docs/adrs/ADR-006-garantia-at-least-once-com-x-event-id.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-006-DEC | docs/adrs/ADR-006-garantia-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID) para dedup no cliente | TRANSCRICAO | [09:25] Diego |
| ADR-006-ALT-01 | docs/adrs/ADR-006-garantia-at-least-once-com-x-event-id.md | Trade-off | Alternativa descartada: exactly-once delivery | TRANSCRICAO | [09:24] Diego |
| ADR-006-ALT-02 | docs/adrs/ADR-006-garantia-at-least-once-com-x-event-id.md | Trade-off | Alternativa descartada: best-effort sem identificador de evento | TRANSCRICAO | [09:25] Diego |
| ADR-006-COD-01 | docs/adrs/ADR-006-garantia-at-least-once-com-x-event-id.md | Restrição | eventId único, padrão UUID do restante do schema | CODIGO | prisma/schema.prisma |

## docs/adrs/ADR-007-payload-snapshot-no-momento-da-insercao.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-007-DEC | docs/adrs/ADR-007-payload-snapshot-no-momento-da-insercao.md | Decisão | Payload serializado como snapshot no momento da inserção | TRANSCRICAO | [09:51] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-payload-snapshot-no-momento-da-insercao.md | Trade-off | Alternativa descartada: lazy rendering no momento do dispatch | TRANSCRICAO | [09:51] Larissa |
| ADR-007-COD-01 | docs/adrs/ADR-007-payload-snapshot-no-momento-da-insercao.md | Restrição | tx.order.findUnique com include já disponível para o snapshot | CODIGO | src/modules/orders/order.service.ts |

## docs/adrs/ADR-008-modulo-webhook-segue-padroes-existentes.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-008-DEC | docs/adrs/ADR-008-modulo-webhook-segue-padroes-existentes.md | Decisão | Módulo webhooks segue controller/service/repository e reaproveita infra | TRANSCRICAO | [09:30] Larissa |
| ADR-008-ALT-01 | docs/adrs/ADR-008-modulo-webhook-segue-padroes-existentes.md | Trade-off | Alternativa descartada: contêiner de IoC (tsyringe/inversify) | TRANSCRICAO | [09:27] Bruno |
| ADR-008-ALT-02 | docs/adrs/ADR-008-modulo-webhook-segue-padroes-existentes.md | Trade-off | Alternativa descartada: EventEmitter/Observer para comunicação entre módulos | TRANSCRICAO | [09:41] Bruno |
| ADR-008-COD-01 | docs/adrs/ADR-008-modulo-webhook-segue-padroes-existentes.md | Restrição | buildControllers(prisma) como raiz de composição | CODIGO | src/app.ts |
| ADR-008-COD-02 | docs/adrs/ADR-008-modulo-webhook-segue-padroes-existentes.md | Restrição | AppError como base para novos códigos WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |

---

## Resumo de cobertura

- Total de itens rastreados: 108
- Itens com Fonte = TRANSCRICAO: 88 (~81%)
- Itens com Fonte = CODIGO: 20 (~19%), todos com caminho de arquivo real do repositório
- Documentos cobertos: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e as 8 ADRs em `docs/adrs/`
