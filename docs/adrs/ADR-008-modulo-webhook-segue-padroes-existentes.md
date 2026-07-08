# ADR-XXX — Módulo de Webhook Segue os Padrões Existentes de Controller→Service→Repository e Reutiliza a Infraestrutura Compartilhada

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma)
- **Revisores:** Larissa (Tech Lead), Sofia (Engenheira de Segurança), Marcos (Product Manager)

---

## Relações

- **Relacionado a:** [ADR-005 — Autenticação HMAC-SHA256 com Secret por Endpoint e Grace Period de Rotação](./ADR-005-autenticacao-hmac-sha256-com-secret-por-endpoint.md)

---

## Contexto

O sistema já possui uma arquitetura modular consolidada em `src/modules/`, onde cada domínio de negócio é organizado em cinco arquivos com responsabilidades bem definidas: `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts`. Esse padrão está em produção nos módulos de usuários (`src/modules/users/`), clientes (`src/modules/customers/`), produtos (`src/modules/products/`), autenticação (`src/modules/auth/`) e pedidos (`src/modules/orders/`). A composição das dependências ocorre de forma centralizada em `src/app.ts`, na função `buildControllers(prisma: PrismaClient)`, que instancia manualmente cada repositório, serviço e controller — injeção de dependência manual sem nenhum contêiner de IoC.

A infraestrutura transversal também já está definida e estável: tratamento de erros centralizado em `src/middlewares/error.middleware.ts` via `errorMiddleware`, autenticação via `authenticate` e controle de perfil via `requireRole` em `src/middlewares/auth.middleware.ts`, validação de entrada via `validate` com schemas Zod em `src/middlewares/validate.middleware.ts`, e logging estruturado com Pino disponível em toda a base de código. A hierarquia de erros parte de `AppError` (em `src/shared/errors/app-error.ts`) e os erros específicos de domínio são subclasses com `errorCode` padronizados — por exemplo `INSUFFICIENT_STOCK` em `InsufficientStockError` e `INVALID_STATUS_TRANSITION` em `InvalidStatusTransitionError`, ambos definidos em `src/shared/errors/http-errors.ts`.

A decisão de implementar o módulo de Webhooks (MOD-008) trouxe a pergunta sobre qual estratégia estrutural adotar. O time, em reunião técnica, avaliou se seria necessário introduzir novos padrões arquiteturais para suportar as particularidades dos webhooks — como processamento assíncrono via worker, gestão de fila de entrega (outbox) e endpoint administrativo de reprocessamento (DLQ replay) — ou se a estrutura existente seria suficiente para acomodar esse novo módulo sem quebrar a coerência da codebase.

---

## Decisão

O módulo de Webhooks (MOD-008) seguirá rigorosamente a mesma estrutura dos módulos existentes, criando os arquivos `src/modules/webhooks/webhook.controller.ts`, `src/modules/webhooks/webhook.service.ts`, `src/modules/webhooks/webhook.repository.ts`, `src/modules/webhooks/webhook.routes.ts` e `src/modules/webhooks/webhook.schemas.ts`. A lógica de processamento assíncrono ficará em `src/modules/webhooks/webhook.processor.ts` (ou `webhook.worker.ts`), chamada a partir de `src/worker.ts` — uma entry-point separada da API, análoga a `src/server.ts`, garantindo que o worker rode em processo independente. O controller de webhooks será adicionado ao retorno da função `buildControllers(prisma)` em `src/app.ts`, sem alterar a mecânica de composição existente.

Para tratamento de erros, serão criadas subclasses de `AppError` com o prefixo `WEBHOOK_` nos códigos de erro — por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` — seguindo exatamente a convenção já estabelecida por `InsufficientStockError` e `InvalidStatusTransitionError`. O `errorMiddleware` capturará essas subclasses automaticamente, sem qualquer modificação, pois já trata genericamente toda instância de `AppError`. Os middlewares `authenticate`, `requireRole` e `validate` serão aplicados nas rotas do módulo sem alterações — o endpoint de replay de DLQ utilizará `requireRole('ADMIN')`, e os endpoints de CRUD de configuração de webhook usarão `authenticate` com qualquer role autenticada. Não será introduzido nenhum novo framework, biblioteca de IoC ou padrão de infraestrutura.

---

## Alternativas Consideradas

### Introdução de Contêiner de IoC ou Framework de Injeção de Dependência

Avaliou-se a adoção de um contêiner de IoC (como `tsyringe` ou `inversify`) para gerenciar as dependências do módulo de webhooks, dado que o worker exige uma instância de `PrismaClient` separada da API. A alternativa foi rejeitada por Bruno e Diego: a injeção de dependência manual via `buildControllers(prisma)` funciona adequadamente para a escala atual do time, e adotar um framework de IoC exigiria refatorar todos os módulos existentes para garantir consistência — um custo desproporcional ao benefício. A separação do `PrismaClient` por processo (API vs. worker) é resolvida de forma simples instanciando uma nova conexão no `src/worker.ts` com a mesma `DATABASE_URL`. `([09:27] Bruno)`

### Event Emitter ou Padrão Observer para Comunicação Interna entre Módulos

Considerou-se o uso de `EventEmitter` nativo do Node.js ou um padrão Observer para desacoplar o módulo de Orders do módulo de Webhooks na hora de publicar eventos de mudança de status. Essa alternativa não foi aprofundada pelo time, que optou pela abordagem de chamada direta de função: `publishWebhookEvent(tx, order, fromStatus, toStatus)`, recebendo o cliente de transação Prisma (`tx`) como parâmetro. Isso garante que a inserção na `webhook_outbox` ocorra dentro da mesma transação SQL da mudança de status do pedido, sem nenhuma dependência de mecanismo de eventos em memória que pudesse ser perdida em caso de falha do processo. `([09:41] Bruno, [09:41] Diego)`

---

## Consequências

### Positivas

- Qualquer desenvolvedor familiarizado com o módulo de pedidos (`src/modules/orders/`) consegue ler, entender e contribuir com o módulo de webhooks imediatamente, sem curva de aprendizado adicional.
- O `errorMiddleware` em `src/middlewares/error.middleware.ts` não requer nenhuma modificação: as subclasses de `AppError` com prefixo `WEBHOOK_` são capturadas automaticamente pelo bloco `if (err instanceof AppError)`.
- Os middlewares `validate` (`src/middlewares/validate.middleware.ts`), `authenticate` e `requireRole` (`src/middlewares/auth.middleware.ts`) são aplicados nas rotas de webhook sem alteração de código ou de interface.
- A função `buildControllers(prisma)` em `src/app.ts` só precisa receber a adição do `WebhookController`, sem refatoração da mecânica de composição.
- Code reviews de PRs do módulo de webhooks seguem o mesmo checklist já usado para os demais módulos, reduzindo o tempo de revisão.
- A separação entre API (`src/server.ts`) e worker (`src/worker.ts`) reutiliza a mesma convenção de entry-point já estabelecida no projeto, sem nova infraestrutura de orquestração.

### Negativas / Trade-offs

- A estrutura Controller→Service→Repository pode ser considerada sobre-engenharia para endpoints administrativos mais simples do módulo de webhooks (como o replay de DLQ), onde toda a lógica poderia estar em uma única camada.
- A convenção de prefixo `WEBHOOK_` para códigos de erro precisa ser documentada explicitamente — sem essa documentação, novos desenvolvedores podem criar subclasses de `AppError` com nomes inconsistentes para o mesmo domínio.
- O worker em processo separado compartilha a mesma `DATABASE_URL` mas instancia um `PrismaClient` independente, o que exige atenção ao gerenciamento de conexões quando o número de processos crescer.
- A ordenação de entrega de eventos é garantida apenas por `order_id` e enquanto o sistema operar com um único worker; escalar para múltiplos workers em paralelo exigirá decisão arquitetural adicional (particionamento por `order_id` ou lock pessimista), conforme limitação registrada em reunião. `([09:13] Diego)`

---

## Referências

### Transcrição

- `[09:27] Bruno` — "a gente não precisa inventar nada novo. Segue o mesmo padrão dos outros módulos."
- `[09:28] Diego` — "controller, service, repository, schemas. Igual."
- `[09:28] Bruno` — "os erros usam o mesmo AppError, só muda o prefixo. WEBHOOK_ alguma coisa."
- `[09:29] Diego` — "o logger é o mesmo Pino. O error middleware já trata AppError, não precisa mudar."
- `[09:30] Larissa` — "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."
- `[09:36] Larissa` — "Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe."
- `[09:41] Bruno` — "Vou propor uma função publishWebhookEvent(tx, order, fromStatus, toStatus) que aceita o tx client da transação atual."
- `[09:41] Diego` — "Boa, função pura recebendo o tx. Não precisa injetar repository inteiro."

### Código

- `src/shared/errors/app-error.ts` — Classe base `AppError` com `statusCode`, `errorCode` e `details`; extensível para subclasses com prefixo `WEBHOOK_`.
- `src/shared/errors/http-errors.ts` — Subclasses existentes (`InsufficientStockError`, `InvalidStatusTransitionError`) que servem de modelo para os novos erros do módulo de webhooks.
- `src/middlewares/error.middleware.ts` — `errorMiddleware` que captura genericamente toda instância de `AppError`, `ZodError` e erros do Prisma; não requer modificação.
- `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole` reutilizados diretamente nas rotas do módulo de webhooks.
- `src/middlewares/validate.middleware.ts` — `validate` aplicado com schemas Zod nas rotas do módulo de webhooks.
- `src/app.ts` — `buildControllers(prisma)` como raiz de composição onde o `WebhookController` será registrado.
- `src/modules/orders/order.controller.ts` — Exemplo de implementação de controller que o módulo de webhooks replica estruturalmente.
- `src/modules/orders/order.service.ts` — Ponto de integração onde `publishWebhookEvent(tx, ...)` será chamado dentro da transação de mudança de status.
