# ADR-XXX — Worker de Webhooks como Processo Node.js Separado

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Diego (Engenheiro Sênior, time de Plataforma), Larissa (Tech Lead)
- **Revisores:** Bruno (Engenheiro Pleno, time de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Contexto

O sistema de notificação de pedidos via webhooks foi concebido para atender a uma demanda formal de clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo), que precisam ser notificados em tempo real — com latência abaixo de 10 segundos — sempre que o status de um pedido muda na plataforma. A solução adotada é o padrão Outbox: dentro da mesma transação SQL que altera o status do pedido, insere-se um evento na tabela `webhook_outbox`. Um componente dedicado, o **worker**, é responsável por ler essa tabela periodicamente e disparar as chamadas HTTP para os endpoints cadastrados pelos clientes.

A questão central desta decisão é: **onde e como o worker deve executar?** Duas abordagens se apresentam naturalmente: (a) embutir o loop de polling diretamente no processo da API Express já existente, ou (b) criar um novo processo Node.js independente. O ponto de partida da discussão foi a constatação de que o Node.js é single-threaded — um loop de polling bloqueante disputaria o event loop com o tratamento de requisições HTTP, degradando a performance da API. Além disso, o ciclo de vida dos dois componentes é fundamentalmente diferente: a API pode ser reiniciada a qualquer momento por deploys ou crashes, enquanto o worker precisa continuar a entrega de webhooks sem interrupção.

O projeto já possui um padrão consolidado de inicialização de processo em `src/server.ts`, que registra handlers de shutdown para `SIGINT` e `SIGTERM` e instancia um `PrismaClient` via a função exportada `createPrismaClient()` de `src/config/database.ts`. Essa função foi deliberadamente desenhada para ser chamada de qualquer ponto de entrada, permitindo que múltiplos processos conectem ao mesmo banco de dados (`DATABASE_URL`) com instâncias de client completamente isoladas. O contexto técnico, portanto, favorecia a criação de um processo separado sem necessidade de grandes mudanças na infraestrutura compartilhada.

## Decisão

O worker de webhooks será implementado como um processo Node.js independente, com entry point em `src/worker.ts`, espelhando o padrão já estabelecido em `src/server.ts`. Ele instanciará seu próprio `PrismaClient` via `createPrismaClient()` (mesma `DATABASE_URL`, sem memória compartilhada com a API), executará um loop de polling a cada 2 segundos buscando eventos pendentes na tabela `webhook_outbox`, e registrará seus próprios handlers de `SIGINT`/`SIGTERM` para encerramento gracioso. A lógica de processamento ficará encapsulada numa classe `WebhookProcessor` dentro do módulo `src/modules/webhooks/`, mantendo o worker slim. O `package.json` receberá um script `"worker"` dedicado para inicializar esse processo separadamente da API.

A separação de processos garante uma fronteira de falha clara: um crash no worker não afeta a disponibilidade da API, e um reinício da API (por deploy ou falha) não interrompe entregas de webhooks em andamento. Ambos os processos compartilham módulos do projeto — `src/shared/logger/`, `src/config/env.ts` e `src/config/database.ts` — via importação estática, sem qualquer canal de comunicação em tempo de execução entre eles. O banco de dados é o único ponto de coordenação: a API escreve na `webhook_outbox`, o worker lê.

## Alternativas Consideradas

### Background task / setInterval dentro do processo da API

A alternativa mais simples seria registrar um `setInterval` dentro do processo Express, chamando periodicamente a lógica de processamento de webhooks. Essa abordagem foi explicitamente rejeitada por Diego em `[09:11]`: um reinício da API mataria qualquer entrega em andamento e o loop de polling disputaria o event loop com o tratamento de requisições HTTP. Além disso, um crash no módulo de webhook poderia derrubar a API inteira, violando o princípio de isolamento de falhas. A frase de Diego foi direta: "o worker tem que ser um processo separado. Se a gente reiniciar a API não pode parar de entregar webhooks."

### Microsserviço em linguagem ou container diferente

A opção de criar um serviço completamente isolado — em outro repositório, outra linguagem ou outro container Docker desacoplado — não chegou a ser formalmente debatida durante a reunião. O time optou por permanecer em Node.js/TypeScript e reaproveitar os módulos compartilhados já existentes (`logger`, `env`, `database`). Subir nova infraestrutura de orquestração para um componente que pode ser co-deployado como segundo processo do mesmo projeto foi considerado overengineering, na mesma linha do raciocínio aplicado ao Redis (`[09:07] Diego`: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering").

## Consequências

### Positivas

- Reinícios da API por deploy, crash ou escalonamento não interrompem entregas de webhooks em andamento, preservando a garantia de at-least-once para os clientes B2B.
- O worker pode ser escalado independentemente da API — se o volume de eventos crescer, é possível aumentar réplicas do worker sem afetar o throughput de HTTP da API.
- Fronteira de falha clara: um crash no worker não derruba a API; um crash na API não para o worker.
- O padrão de `src/server.ts` — `bootstrap()`, handlers de `SIGINT`/`SIGTERM`, `prisma.$disconnect()` — é integralmente reaproveitado em `src/worker.ts`, minimizando a curva de aprendizado da equipe.
- O uso de `createPrismaClient()` exportado de `src/config/database.ts` já suporta múltiplas instâncias por design, sem nenhuma mudança necessária nesse módulo.
- Módulos compartilhados (`src/shared/logger/`, `src/config/env.ts`) funcionam igualmente nos dois processos, mantendo consistência de logging e configuração.

### Negativas / Trade-offs

- Dois processos para fazer deploy, monitorar e reiniciar em caso de falha — o `package.json` precisa de um script `"worker"` e a infraestrutura de deploy precisa gerenciar dois processos distintos.
- Dois `PrismaClient` ativos simultaneamente, cada um com seu próprio pool de conexões, aumentando o número de conexões abertas com o MySQL.
- Módulos compartilhados (`src/shared/logger/`, `src/config/env.ts`, `src/config/database.ts`) devem ser mantidos sem side effects na importação, uma restrição que não existia enquanto havia apenas um processo na aplicação.
- Ausência de garantia de ordering global entre eventos: o single-worker atual entrega eventos na ordem de `created_at` da `webhook_outbox`, mas qualquer futura escala horizontal do worker quebraria essa garantia — limitação reconhecida e registrada em `[09:13] Diego` como "problema do futuro".
- Não há canal de comunicação direta entre API e worker para situações de emergência (ex.: pausar o worker remotamente); toda coordenação passa pelo banco de dados.

## Referências

### Transcrição

- `[09:11] Diego` — "o worker tem que ser um processo separado. Se a gente reiniciar a API não pode parar de entregar webhooks"
- `[09:11] Larissa` — "Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um script 'npm run worker'"
- `[09:11] Bruno` — "Pode ser, mas vai precisar conectar no mesmo banco e usar o mesmo Prisma client"
- `[09:11] Diego` — "Sim, mesmo banco, mesma stack. Só não pode ser o mesmo processo"
- `[09:28] Bruno` — "Eu colocaria src/worker.ts como entry separada, e a lógica de processamento fica num arquivo dentro do módulo, tipo src/modules/webhooks/webhook.worker.ts ou webhook.processor.ts"
- `[09:29] Bruno` — "o worker importa o logger do shared, mesma config"
- `[09:29] Diego` — "Sobre infraestrutura compartilhada: o pool de conexão do Prisma já tá lá. O worker abre o mesmo PrismaClient ou um separado?"
- `[09:30] Bruno` — "Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node"
- `[09:07] Diego` — "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering"
- `[09:13] Diego` — "Se a gente escala pra múltiplos workers em paralelo no futuro, perde a garantia [de ordering]. Por enquanto, single-worker e ordering implícita por order_id"

### Código

- `src/server.ts` — entry point existente da API; padrão de `bootstrap()`, `SIGINT`/`SIGTERM` e `prisma.$disconnect()` que `src/worker.ts` irá espelhar
- `src/config/database.ts` — exporta `createPrismaClient()`, permitindo instanciar um `PrismaClient` independente por processo sem alterações
- `src/worker.ts` — entry point do worker (a ser criado)
- `src/modules/webhooks/webhook.processor.ts` — classe `WebhookProcessor` com a lógica de polling e dispatch (a ser criada)
- `src/shared/logger/index.js` — logger Pino compartilhado; importado sem modificações pelo worker
- `src/config/env.ts` — configurações de ambiente compartilhadas; importadas sem modificações pelo worker
