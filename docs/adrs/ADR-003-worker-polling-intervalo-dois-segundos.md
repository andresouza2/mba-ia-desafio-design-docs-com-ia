# ADR-XXX — Worker de Webhooks com Polling de 2 Segundos na webhook_outbox

- **Status:** Aceito
- **Data:** 2026-07-06
- **Autores:** Diego (Engenheiro Sênior, time de Plataforma), Larissa (Tech Lead)
- **Revisores:** Bruno (Engenheiro Pleno, time de Pedidos), Marcos (Product Manager), Sofia (Engenheira de Segurança)

---

## Relações

- **Depende de:** [ADR-001 — Uso do Padrão Transactional Outbox no MySQL para Captura Atômica de Eventos de Webhook](./ADR-001-transactional-outbox-pattern-in-mysql.md)
- **Depende de:** [ADR-002 — Worker de Webhooks como Processo Node.js Separado](./ADR-002-worker-como-processo-nodejs-separado.md)
- **Relacionado a:** [ADR-004 — Política de Retry com Backoff Exponencial e Dead Letter Queue para Webhooks](./ADR-004-politica-de-retry-com-backoff-exponencial-e-dlq.md)
- **Alterado por:** [ADR-007 — Payload da Outbox Armazenado como Snapshot no Momento da Inserção](./ADR-007-payload-snapshot-no-momento-da-insercao.md)

---

## Contexto

O sistema de webhooks adota o padrão Transactional Outbox para garantir entrega confiável de eventos de mudança de status de pedidos. Dentro da mesma transação SQL que atualiza a tabela `orders` e insere registros em `order_status_history`, uma linha é gravada na tabela `webhook_outbox` com o payload do evento e o status inicial `PENDING`. Um worker independente — executado como processo separado em `src/worker.ts`, com entrada de processamento em `src/modules/webhooks/webhook.processor.ts` — é o responsável por ler essa tabela e despachar as chamadas HTTP para os endpoints dos clientes.

Para que o worker detecte novos eventos na `webhook_outbox`, é necessário um mecanismo de acionamento. A abordagem ideal seria um modelo reativo, em que o banco de dados notificasse o worker imediatamente ao inserir um novo registro. Contudo, o MySQL — banco de dados utilizado pelo projeto, conforme definido em `prisma/schema.prisma` — não oferece um mecanismo nativo equivalente ao `NOTIFY/LISTEN` do PostgreSQL para comunicação com processos externos. Como registrado na transcrição da reunião técnica, `[09:10] Larissa` confirmou: *"MySQL não tem NOTIFY nativo pra aplicação externa. Polling é o caminho."* Qualquer solução baseada em triggers do banco para notificar o worker exigiria infraestrutura adicional não prevista no escopo da fase atual.

O requisito de negócio, levantado pelos clientes B2B Atlas Comercial, MaxDistribuição e Nova Cargo, estabelece que qualquer entrega de notificação abaixo de 10 segundos é considerada "tempo real" para fins práticos, conforme relatado por Marcos em `[09:02]`. Essa folga de até 10 segundos entre a mudança de status e a entrega do webhook abre espaço para uma abordagem de polling com intervalo curto, sem necessidade de mecanismos mais complexos ou infraestrutura adicional.

---

## Decisão

O worker de webhooks realizará polling ativo na tabela `webhook_outbox` a cada **2 segundos**, buscando linhas com `status = PENDING` e `nextRetryAt <= NOW()`, ordenadas por `createdAt ASC` para garantir processamento na ordem de chegada dos eventos — o que preserva a ordem de entrega por `order_id` enquanto o sistema operar com um único worker, conforme acordado em `[09:12] Diego` e `[09:13] Larissa`. Esse intervalo foi deliberadamente escolhido pela equipe como o ponto de equilíbrio entre reatividade e carga no banco: curto o suficiente para respeitar o SLA de menos de 10 segundos (pior caso: 2 s de polling + latência de rede), e longo o suficiente para não sobrecarregar o MySQL com queries contínuas, como afirmou Diego em `[09:10]`: *"dois segundos é razoável. Não sobrecarrega o banco."*

Para que as queries de polling sejam eficientes à medida que a tabela `webhook_outbox` cresce, são necessários índices compostos sobre os campos consultados na cláusula `WHERE` e `ORDER BY`. Os índices mínimos exigidos são: `(status, nextRetryAt)` — para filtrar rapidamente as linhas elegíveis para processamento — e `(createdAt)` — para ordenação eficiente. Sem esses índices, uma query de polling num cenário com muitos registros históricos (entregues, com falha, em backoff) resultaria em varredura completa da tabela a cada ciclo de 2 segundos, degradando progressivamente a performance do banco. A função `createPrismaClient()` definida em `src/config/database.ts` é projetada para ser instanciada de forma independente, permitindo que o worker mantenha seu próprio pool de conexões sem compartilhar estado com a API principal.

---

## Alternativas Consideradas

### MySQL triggers com notificação externa

A ideia de usar triggers do MySQL para reagir a inserções na `webhook_outbox` e disparar o worker de forma imediata foi levantada por Bruno em `[09:09]`: *"Não dá pra usar trigger do banco pra ser mais reativo?"*. Diego rejeitou a abordagem em `[09:09]`: triggers do MySQL executam apenas SQL interno e não possuem mecanismo nativo para notificar processos externos. Para que um trigger pudesse sinalizar o worker, seria necessário recorrer a soluções improvisadas — como escrever em um arquivo compartilhado ou chamar um endpoint HTTP via UDF (User-Defined Function) — o que introduziria complexidade operacional, pontos de falha adicionais e dependências de infraestrutura fora do escopo atual. A avaliação da equipe foi que a solução ficaria *"esquisita"* e não justificaria a complexidade em relação ao polling simples.

### Fila baseada em Redis (Redis Streams)

A utilização de Redis Streams como mecanismo de enfileiramento e notificação imediata foi mencionada por Larissa em `[09:07]`: *"A alternativa seria botar Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra."* Diego reforçou a rejeição em `[09:07]`: *"Exato, e a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve."* O Redis não faz parte da infraestrutura atual do projeto, e introduzi-lo apenas para servir como canal de notificação do worker adicionaria custo operacional, complexidade de provisionamento e um novo ponto de falha, sem benefício proporcional dado o SLA de 10 segundos já satisfeito pelo polling.

### Intervalo de polling mais curto (ex.: 500 ms)

Um intervalo significativamente menor, como 500 ms, aumentaria a reatividade mas multiplicaria por quatro o volume de queries ao banco em relação à solução de 2 segundos. A equipe não debateu explicitamente essa alternativa durante a reunião — o intervalo de 2 segundos foi proposto por Diego e aceito sem contestação, indicando consenso de que ele já era suficientemente curto para os requisitos do negócio.

---

## Consequências

### Positivas

- Implementação simples: um loop `setInterval` no worker com uma query Prisma, sem dependências externas além do MySQL já existente.
- Satisfaz plenamente o SLA de entrega abaixo de 10 segundos: no pior caso, um evento aguarda 2 segundos até o próximo ciclo de polling, mais o tempo de rede.
- Carga previsível e controlada no banco de dados: uma query indexada a cada 2 segundos por ciclo de worker.
- Nenhuma infraestrutura adicional é necessária: o worker conecta-se ao mesmo banco MySQL via uma instância independente de `PrismaClient`, conforme o padrão já estabelecido em `src/config/database.ts`.
- Ordering implícita por `createdAt ASC` garante que eventos de um mesmo pedido sejam entregues na sequência correta enquanto o sistema operar com worker único.
- Facilidade de observabilidade: o estado de cada evento (`PENDING`, em backoff, `FAILED`) é visível diretamente na tabela `webhook_outbox`, sem necessidade de ferramentas externas.

### Negativas / Trade-offs

- Latência mínima de processamento de **até 2 segundos** após a inserção do evento na `webhook_outbox`, mesmo que o worker esteja ocioso. Eventos críticos não são despachados instantaneamente.
- Sob alto volume de mudanças de status simultâneas (ex.: 50 pedidos mudando em 1 minuto, ponto registrado por Diego em `[09:38]`), um worker single-threaded processando um evento por ciclo pode acumular fila. Processamento em batch ou workers paralelos poderão ser necessários no futuro — decisão explicitamente adiada pela equipe.
- A escalabilidade horizontal para múltiplos workers paralelos **quebra a garantia de ordering**: conforme alertado por Diego em `[09:12]`, com múltiplos workers a ordem de entrega por `order_id` deixa de ser garantida. A solução futura envolveria particionamento por `order_id` ou uso de locks pessimistas — registrado como limitação conhecida em `[09:13] Larissa`.
- Os índices `(status, nextRetryAt)` e `(createdAt)` na tabela `webhook_outbox` são **pré-requisitos de performance** para o polling, conforme identificado no arquivo de potencial ADR (`docs/adrs/potential-adrs/must-document/MOD-008/worker-polling-interval-two-seconds.md`). A ausência desses índices degradará progressivamente o banco conforme o volume de eventos históricos cresce — especialmente linhas com status `DELIVERED` ou `FAILED` que não são limpas imediatamente (arquivamento após 30 dias foi mencionado por Diego em `[09:08]` como escopo futuro).
- O worker precisa rodar como **processo separado** da API (`src/worker.ts` vs. `src/server.ts`), com script dedicado `npm run worker`, conforme decidido em `[09:11] Diego` e `[09:11] Larissa`. Reinicializações da API não afetam o worker, mas também exigem gestão de ciclo de vida independente em produção.

---

## Referências

### Transcrição

- `[09:02] Marcos` — requisito de SLA: notificações abaixo de 10 segundos são aceitas como "tempo real" pelos clientes B2B.
- `[09:08] Diego` — menção a índices em `status` e `created_at` na tabela de outbox para eficiência do polling.
- `[09:09] Diego` — definição do mecanismo: *"Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca."*
- `[09:09] Bruno` — questionamento sobre uso de triggers do banco como alternativa reativa.
- `[09:09] Diego` — rejeição de triggers: MySQL não notifica processos externos nativamente; polling de 2 s atende o SLA.
- `[09:10] Larissa` — confirmação: *"MySQL não tem NOTIFY nativo pra aplicação externa. Polling é o caminho."*
- `[09:10] Diego` — *"dois segundos é razoável. Não sobrecarrega o banco."*
- `[09:10] Marcos` — *"2 segundos serve, perfeito."*
- `[09:10] Larissa` — registro formal da decisão: *"Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos."*
- `[09:11] Diego` — worker como processo separado, não dentro da mesma instância da API.
- `[09:11] Larissa` — proposta de entry-point `src/worker.ts` e script `npm run worker`.
- `[09:12] Diego` — ordering implícita por `createdAt` com worker único; perda de garantia com múltiplos workers.
- `[09:13] Larissa` — limitação conhecida documentada: sem garantia de ordering global.
- `[09:07] Diego` — rejeição de Redis Streams: overengineering para time pequeno sem infraestrutura Redis.
- `[09:38] Diego` — rate limiting e volume alto de eventos como ponto em aberto a ser observado.

### Código

- `C:\www\full-cycle\mba\mba-ia-desafio-design-docs-com-ia\src\config\database.ts` — função `createPrismaClient()` projetada para instanciação independente por processo, habilitando o worker a ter seu próprio pool de conexões.
- `C:\www\full-cycle\mba\mba-ia-desafio-design-docs-com-ia\prisma\schema.prisma` — definição do banco MySQL como provider (`datasource db`), confirmando ausência de suporte nativo a `NOTIFY/LISTEN`; padrão de índices do projeto (ex.: `@@index([status])` em `orders`, `@@index([createdAt])`) que deve ser replicado em `webhook_outbox`.
- `C:\www\full-cycle\mba\mba-ia-desafio-design-docs-com-ia\docs\adrs\potential-adrs\must-document\MOD-008\worker-polling-interval-two-seconds.md` — arquivo de potencial ADR com evidências coletadas, score de priorização e resumo da decisão.
